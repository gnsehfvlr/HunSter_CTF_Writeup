# [RedPwn 2026] dimensionality

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Hard |
| **Points** | Unknown |

---

## 📌 개요

`dimensionality`는 3D 미로 탐색과 RC4 기반 복호화를 결합한 리버싱 문제입니다. 사용자는 주어진 바이너리에서 숨겨진 플래그를 찾기 위해 정확한 경로를 계산하고, 이를 키로 사용해 암호화된 데이터를 복호화해야 합니다.

- **Target:** ELF 64-bit, stripped, PIE 바이너리 (`chall`)
- **핵심 포인트:** 3D 그리드 기반 미로 탐색 + 사용자 입력을 키로 한 RC4 복호화

---

## 🔍 정찰

### 1) 바이너리 기본 정보 확인
먼저 바이너리의 기본 정보를 확인하고, 문자열을 추출하여 초기 단서를 수집합니다.

```bash
file /sandbox/bins/chall
strings /sandbox/bins/chall | grep -v '/lib\|GLIBC\|SUITE\|GCC\|crt\|deregister'
```

**관찰 결과:**
- `ELF 64-bit LSB pie executable`, stripped → 분석 난이도 상승
- `fgets`, `puts`, `fwrite` 사용 → 입력을 받고 출력하는 구조
- `~ !"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstuvwxyz{|}~` → 인코딩/문자셋 후보
- `:(` 출력 → 잘못된 입력에 대한 실패 메시지

### 2) 동작 테스트
테스트 입력을 제공하여 프로그램의 동작을 확인합니다.

```bash
echo 'AAAA' | /sandbox/bins/chall
```

**출력:** `:(`  
→ 입력 검증 로직 존재, 성공 시 다른 출력(`:)`)이 있을 것으로 추정

---

## 🧠 문제 분석

### 1) Radare2를 이용한 정적 분석
`radare2`로 함수 목록을 확인하고, `main`과 핵심 함수를 디컴파일합니다.

```bash
r2 -A /sandbox/bins/chall
> afl
> pdf @main
```

**주요 함수:**
- `main`: 입력 받고 `fcn.00001410`로 검증
- `fcn.00001410`: 입력 문자열 기반 3D 그리드 탐색
- `fcn.000012b0`: RC4 KSA(Key Scheduling Algorithm) 구현

### 2) 3D 미로 구조 분석
`fcn.00001410`의 디컴파일 결과를 분석하면 다음과 같은 구조를 확인할 수 있습니다:

- 그리드 크기: `*0x408c` → 런타임에 결정 (실제 값: `0x0b` = 11)
- 그리드 시작 주소: `0x2080`
- 시작 위치: 값이 `0x02`인 셀
- 목표 위치: 값이 `0x03`인 셀
- 이동 명령어:
  - `'b'`: -z
  - `'d'`: +y
  - `'f'`: +z
  - `'l'`: -x
  - `'r'`: +x
  - `'u'`: -y

### 3) 그리드 데이터 추출
`.rodata` 섹션에서 그리드 데이터를 추출합니다.

```python
import lief

binary = lief.parse('/sandbox/bins/chall')
for section in binary.sections:
    if section.name == '.rodata':
        grid_data = section.content[0x80:0x80+1331]  # 11x11x11 = 1331 bytes
        break

def to_3d(idx, size=11):
    x = idx % size
    y = (idx // size) % size
    z = idx // (size * size)
    return (x, y, z)

# 시작(2)과 끝(3) 위치 찾기
start = end = None
for i, val in enumerate(grid_data):
    if val == 2:
        start = to_3d(i)
    elif val == 3:
        end = to_3d(i)

print(f"Start: {start}, End: {end}")
```

**출력:** `Start: (7, 7, 0), End: (9, 7, 10)`

### 4) 경로 탐색 (BFS)
시작점에서 목표점까지의 최단 경로를 탐색합니다. 장애물(값이 1인 셀)과 값이 0인 셀은 이동 불가.

```python
from collections import deque

def bfs_path(grid_data, start, end, size=11):
    def to_1d(x, y, z): return z*size*size + y*size + x
    def is_valid(x, y, z):
        if not (0 <= x < size and 0 <= y < size and 0 <= z < size):
            return False
        val = grid_data[to_1d(x, y, z)]
        return val != 1 and val != 0  # 장애물 및 빈 공간 회피

    q = deque([(start, "")])
    visited = set()
    visited.add(start)

    moves = {
        (0, 0, 1): 'f',
        (0, 0, -1): 'b',
        (1, 0, 0): 'r',
        (-1, 0, 0): 'l',
        (0, 1, 0): 'd',
        (0, -1, 0): 'u'
    }

    while q:
        (x, y, z), path = q.popleft()
        if (x, y, z) == end:
            return path
        for (dx, dy, dz), char in moves.items():
            nx, ny, nz = x+dx, y+dy, z+dz
            if (nx, ny, nz) not in visited and is_valid(nx, ny, nz):
                visited.add((nx, ny, nz))
                q.append(((nx, ny, nz), path + char))
    return None

path = bfs_path(grid_data, start, end)
print(f"Path: {path}")
```

**결과:** `fddllllffrrffffrrffuubbrrfff`

---

## 💥 익스플로잇

### 1) 경로로 바이너리 실행
정답 경로를 입력으로 제공하여 성공 메시지 확인:

```bash
echo 'fddllllffrrffffrrffuubbrrfff' | /sandbox/bins/chall
```

**출력:**  
```
:)
star_/_so_bright_/_car_/_site_-ppsu
```

→ `:)` 이후 실제 플래그 출력

### 2) 플래그 추출
출력에서 `:)` 이후의 문자열을 플래그로 제출합니다.

---

## 🚩 Flag

```
flag{star_/_so_bright_/_car_/_site_-ppsu}