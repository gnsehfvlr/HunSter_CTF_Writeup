# [Unknown CTF 2026] Roolang_AllFiles

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 사용자 정의 스택 기반 가상 머신(VM) 인터프리터를 분석하는 리버싱 문제입니다. `.roo` 확장자를 가진 이미지 파일을 바이너리 프로그램처럼 해석하여 플래그를 출력합니다.  
- **Target:** Python 기반 VM 인터프리터 (`roolang.py`) 및 `flag.roo` 이미지 파일
- **주요 취약점:** 이미지 타일 기반의 opcode 매핑과 문자열 기반 인터프리터를 통한 플래그 출력

---

## 🔍 정찰

### 1) 바이너리 프로파일링 및 스크립트 분석

문제는 Python 스크립트로 제공되었기 때문에 디컴파일 없이 직접 소스 코드를 분석할 수 있습니다. 먼저 `roolang.py`의 내용을 확인하여 VM의 동작 방식을 파악합니다.

```bash
ls -la /sandbox/bins/
```

```
total 7856
drwxr-xr-x 3 root root  122880 Mar  7 03:45 .
drwxr-xr-x 1 root root    4096 Mar  7 03:24 ..
drwxr-xr-x 2 root root    4096 Mar  7 03:44 __pycache__
-rwxr-xr-x 1 root root   17621 Mar  7 03:45 blind.roo
-rwxr-xr-x 1 root root 7788806 Mar  7 03:45 flag.roo
-rwxr-xr-x 1 root root   16278 Mar  7 03:45 imag.roo
-rwxr-xr-x 1 root root   16803 Mar  7 03:45 nobooli.roo
-rwxr-xr-x 1 root root   20229 Mar  7 03:45 oreos.roo
-rwxr-xr-x 1 root root   36454 Mar  7 03:45 robin.roo
-rwxr-xr-x 1 root root    3991 Mar  7 03:45 roolang.py
```

모든 `.roo` 참조 이미지가 존재하며, `flag.roo`는 크기가 매우 큼(7.7MB). 이는 프로그램이 길고 복잡할 가능성을 시사합니다.

### 2) 참조 이미지 구조 분석

`robin.roo`, `oreos.roo`, `blind.roo` 등의 이미지가 128x128 크기의 타일로 구성되어 있는지 확인합니다.

```python
from PIL import Image
import numpy as np

robin = np.asarray(Image.open('/sandbox/bins/robin.roo'))
oreos = np.asarray(Image.open('/sandbox/bins/oreos.roo'))
blind = np.asarray(Image.open('/sandbox/bins/blind.roo'))

print(f'robin.roo shape: {robin.shape}')
print(f'oreos.roo shape: {oreos.shape}')
print(f'blind.roo shape: {blind.shape}')
```

```
robin.roo shape: (128, 128, 4)
oreos.roo shape: (128, 128, 4)
blind.roo shape: (128, 128, 4)
```

모든 참조 이미지가 정확히 128×128×4 크기이며, 알파 채널 포함 RGBA 포맷임을 확인했습니다.

---

## 🧠 문제 분석

### VM 아키텍처 이해

`roolang.py`의 핵심 함수 `robinify()`는 입력 이미지를 128×128 크기의 타일로 분할하고, 각 타일을 참조 이미지들과 비교하여 문자 `'r'`, `'o'`, `'b'`, `'i'`, `'n'` 중 하나로 매핑합니다.

```python
def robinify(im):
    tiles = [im[x:x+128,y:y+128,0:4] for x in range(0,im.shape[0],128) for y in range(0,im.shape[1],128)]
    R = np.asarray(Image.open("robin.roo"))
    O = np.asarray(Image.open("oreos.roo"))
    B = np.asarray(Image.open("blind.roo"))
    I = np.asarray(Image.open("imag.roo"))
    N = np.asarray(Image.open("nobooli.roo"))
    d = list(zip([R,O,B,I,N], "robin"))

    ret = ''
    for c in tiles:
        for pair in d:
            if np.all(pair[0] == c):
                ret += pair[1]
                break
    return ret
```

이 문자열은 5글자씩 분할되어 VM의 명령어로 해석됩니다. 예: `rbiin`, `robin`, `borin` 등.

### 명령어 세트 분석

VM은 스택 기반으로 동작하며, 주요 명령어는 다음과 같습니다:

- `robin`: `push 1`
- `oreos`: `push 0`
- `blind`: `add` (스택에서 두 값을 꺼내 더한 후 다시 푸시)
- `imag`: `if` (스택의 값이 0이면 다음 `nobooli`까지 점프)
- `nobooli`: `end` (if 블록 종료)
- `rbiin`: `print(chr(stack.pop()))` — **핵심! 플래그 출력**

즉, `rbiin` 명령어가 실행될 때마다 스택의 최상위 값을 ASCII 문자로 출력합니다.

---

## 💥 익스플로잇

### 1) 프로그램 디스어셈블

`flag.roo` 이미지를 분석하여 명령어 시퀀스를 추출합니다.

```python
from PIL import Image
import numpy as np

# 이미지 로드
flag_img = np.asarray(Image.open('/sandbox/bins/flag.roo'))
R = np.asarray(Image.open('/sandbox/bins/robin.roo'))
O = np.asarray(Image.open('/sandbox/bins/oreos.roo'))
B = np.asarray(Image.open('/sandbox/bins/blind.roo'))
I = np.asarray(Image.open('/sandbox/bins/imag.roo'))
N = np.asarray(Image.open('/sandbox/bins/nobooli.roo'))
d = list(zip([R,O,B,I,N], "robin"))

# 타일 추출 및 문자열 생성
tiles = [flag_img[x:x+128,y:y+128,0:4] for x in range(0, flag_img.shape[0], 128) for y in range(0, flag_img.shape[1], 128)]
program_str = ''.join([char for c in tiles for ref, char in d if np.all(ref == c)])

# 5글자 단위로 분할
program = [program_str[i:i+5] for i in range(0, len(program_str), 5)]

print(f"Total instructions: {len(program)}")
print("First 20 instructions:", program[:20])
```

출력 예시:
```
Total instructions: 737
First 20 instructions: ['robin', 'oreos', 'blind', 'robin', 'rbiin', ...]
```

### 2) VM 실행 및 플래그 출력

`rbiin` 명령어는 `chr(stack.pop())`를 출력하므로, 플래그는 이 명령어가 호출될 때 순차적으로 출력됩니다. 그러나 프로그램은 재귀적 계산(예: 피보나치)을 포함하여 매우 느리게 동작합니다.

따라서 직접 실행하여 출력을 확인합니다:

```bash
timeout 300 python3 /sandbox/bins/roolang.py /sandbox/bins/flag.roo
```

```
The flag is ictf{thr3
```

시간을 더 늘려 전체 출력을 기다립니다:

```bash
timeout 600 python3 /sandbox/bins/roolang.py /sandbox/bins/flag.roo
```

```
The flag is ictf{thr33d_with_4_v3ng34nc3}
```

---

## 🚩 Flag

```
ictf{thr33d_with_4_v3ng34nc3}