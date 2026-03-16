# [picoCTF 2026] hurry-up!-wait!

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 `.exe` 확장자를 가졌지만 실제로는 **Linux용 ELF 바이너리**이며, **Ada 언어로 컴파일된** `svchost.exe` 파일을 역분석하여 숨겨진 플래그를 찾는 리버싱 문제입니다.  
바이너리는 문자열을 동적으로 조합하는 방식으로 플래그를 출력하며, 여러 함수 호출을 통해 문자 하나씩을 조합합니다.

- **Target:** ELF 64-bit, Ada 컴파일 바이너리 (`svchost.exe`)
- **핵심 포인트:** `LEA` 명령어를 통해 문자열 주소를 계산하고, 특정 오프셋에서 문자를 추출하여 플래그를 구성

---

## 🔍 정찰

### 1) 파일 정보 확인 및 문자열 추출

먼저 제공된 바이너리의 종류를 확인하기 위해 `file` 명령어를 사용합니다. 확장자가 `.exe`이지만, 실제로는 **Linux ELF 바이너리**임을 확인합니다.

```bash
file /sandbox/bins/svchost.exe
```

**관찰 결과:**
```
/sandbox/bins/svchost.exe: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, ...
```

이후 `strings` 명령어로 바이너리 내부의 문자열을 추출합니다.

```bash
strings /sandbox/bins/svchost.exe
```

**관찰 결과:**
- `libgnat-7.so.1` → Ada 컴파일러(GNAT) 사용
- `send_secret_1`, `send_secret_2`, `send_secret_3` → 플래그 관련 함수 추정
- `CTF_{}` → 플래그 포맷 힌트 (실제로는 `flag{}` 형식 사용)

### 2) Radare2를 이용한 정적 분석

Radare2를 사용해 함수 목록을 분석하고, 플래그 생성과 관련된 함수를 탐색합니다.

```bash
r2 -A /sandbox/bins/svchost.exe
```

```r2
afl~send_secret
```

**관찰 결과:**
- `send_secret` 관련 함수 존재
- `fcn.0000298a` 함수가 여러 `send` 함수를 호출하는 것으로 보아 플래그 생성 로직 중심

---

## 🧠 문제 분석

### 1) 핵심 함수 분석 (`fcn.0000298a`)

`fcn.0000298a` 함수는 총 **26개의 함수를 순차적으로 호출**하며, 각 함수는 `ada__text_io__put__4`를 통해 문자열을 출력합니다.  
이들 함수는 모두 `LEA` 명령어를 사용해 `.rodata` 섹션의 특정 문자열 주소를 로드합니다.

예시: `fcn.000024aa` 분석

```r2
pdf @ fcn.000024aa
```

**관찰 결과:**
```
lea rax, [0x00002cd1]  ; 문자열 "ijklmnopqrstuvwxyzCTF_{}" 주소
mov rdi, rax
call sym.ada__text_io__put__4
```

이 문자열은 **기준 문자열의 일부분**입니다. 전체 기준 문자열은 다음과 같습니다:

```
"123456789abcdefghijklmnopqrstuvwxyzCTF_{}"
```

이 문자열의 시작 주소는 `0x2cc0`이며, 각 함수는 이 문자열의 **다른 오프셋에서 시작하는 부분 문자열**을 출력합니다.

### 2) 문자열 오프셋 계산 방식

`LEA` 명령어는 **RIP-상대 주소**(RIP-relative addressing)를 사용합니다.  
예: `48 8d 05 17 08 00 00` → `LEA rax, [rip + 0x817]`

- `LEA` 명령어 위치: `0x24b3`
- 다음 명령어 주소: `0x24b3 + 7 = 0x24ba`
- 실제 문자열 주소: `0x24ba + 0x817 = 0x2cd1`

이 값을 기준 문자열 주소 `0x2cc0`에서 빼면 오프셋을 계산할 수 있습니다:

```
0x2cd1 - 0x2cc0 = 0x11 = 17 → 문자 'i'
```

### 3) 플래그 구성 로직

각 함수가 출력하는 문자열의 **첫 번째 문자**를 추출하면, 플래그를 구성할 수 있습니다.  
`fcn.0000298a`의 **호출 순서**에 따라 문자를 추출하면 플래그가 완성됩니다.

---

## 💥 익스플로잇

다음 Python 스크립트를 사용해 자동으로 플래그를 추출할 수 있습니다.

```python
import struct

# 바이너리 파일 읽기
with open('/sandbox/bins/svchost.exe', 'rb') as f:
    data = f.read()

# 기준 문자열
base_string = "123456789abcdefghijklmnopqrstuvwxyzCTF_{}"
base_addr = 0x2cc0

# fcn.0000298a 내부의 호출 순서 (수동 분석 또는 자동 추출)
call_sequence = [
    0x24aa, 0x2372, 0x25e2, 0x2852, 0x2886, 0x28ba, 0x2922,
    0x23a6, 0x2136, 0x2206, 0x230a, 0x2206, 0x257a, 0x28ee,
    0x240e, 0x26e6, 0x2782, 0x28ee, 0x23a6, 0x240e, 0x233e,
    0x23a6, 0x2372, 0x2206, 0x23a6, 0x2956
]

flag_chars = []

for func_addr in call_sequence:
    # 함수 내 LEA 명령어 찾기 (48 8d 05 xx xx xx xx)
    offset = func_addr
    lea_pos = data.find(b'\x48\x8d\x05', offset, offset + 0x50)
    if lea_pos == -1:
        continue

    # 상대 오프셋 추출 (little-endian)
    rel_offset = struct.unpack('<i', data[lea_pos + 3:lea_pos + 7])[0]
    next_instr = lea_pos + 7
    string_addr = next_instr + rel_offset

    # 문자열 오프셋 계산
    char_offset = string_addr - base_addr
    if 0 <= char_offset < len(base_string):
        flag_chars.append(base_string[char_offset])

# 플래그 조합
flag = ''.join(flag_chars)
print("Reconstructed flag:", flag)
```

**실행 결과:**
```
Reconstructed flag: icoCTF{d15a5m_ftw_dfbdc5d}
```

하지만 문제의 플래그 포맷은 `flag{...}`입니다.  
`icoCTF{...}` → `flag{...}`로 변환하면 최종 플래그가 됩니다.

---

## 🚩 Flag

```
flag{d15a5m_ftw_dfbdc5d}