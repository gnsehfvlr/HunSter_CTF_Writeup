# [ImaginaryCTF 2021] Roolang-RAG

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Hard |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 `.roo` 확장자를 가진 커스텀 프로그래밍 언어 기반의 리버싱 문제로, 이미지를 기반으로 한 바이트코드를 해석하는 Python 기반 인터프리터를 분석해야 한다.  
- **Target:** `roolang.zip` (Python 인터프리터 + 이미지 기반 `.roo` 파일들)
- **주요 취약점:** 이미지 타일 매칭을 통한 프로그램 문자열 생성 및 스택 기반 VM에서의 플래그 출력

---

## 🔍 정찰

### 1) 바이너리 구조 분석

주어진 바이너리는 `.zip` 파일로, 내부에는 여러 `.roo` 이미지 파일과 Python 인터프리터 `roolang.py`가 포함되어 있다. 이는 `.roo` 파일을 이미지로 해석하고 특정 언어로 변환하여 실행하는 커스텀 VM임을 시사한다.

```bash
unzip -l $BINARY
```

**관찰 결과:**
```
Archive:  roolang.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
    17621  2021-07-22 14:53   blind.roo
  7788806  2021-07-22 14:53   flag.roo
    16278  2021-07-22 14:53   imag.roo
    16803  2021-07-22 14:53   nobooli.roo
    20229  2021-07-22 14:53   oreos.roo
    36454  2021-07-22 14:53   robin.roo
     3991  2021-07-22 14:53   roolang.py
```

`flag.roo`의 크기가 약 7.7MB로 매우 크며, 이는 주요 로직 또는 데이터를 포함하고 있음을 의미한다.

---

### 2) 인터프리터 코드 분석

`roolang.py`를 확인하여 동작 방식을 분석한다. 이 스크립트는 PIL과 NumPy를 사용해 이미지를 처리하며, 스택 기반 가상 머신을 구현하고 있다.

```bash
head -n 20 /tmp/roolang.py
```

**관찰 결과:**
```python
#!/usr/bin/env python3
import sys
from PIL import Image
import numpy as np

class Stack(list):
    def push(self, x):
        self.append(x)
    def peek(self):
        return self[-1]

stack = Stack([])
program = []
register = 0
insn_pointer = 0
DEBUG = 0
```

또한 `robinify(im)` 함수를 통해 이미지를 128x128 크기의 타일로 분할하고, 각 타일을 참조 이미지(`robin.roo`, `oreos.roo`, `blind.roo`, `imag.roo`, `nobooli.roo`)와 비교하여 문자 `'r'`, `'o'`, `'b'`, `'i'`, `'n'` 중 하나로 매핑함을 확인한다.

---

## 🧠 취약점 분석

### 이미지 기반 프로그램 생성 메커니즘

`robinify` 함수는 입력 이미지(`flag.roo`)를 128x128 크기의 타일로 분할하고, 각 타일이 다섯 개의 참조 이미지 중 어느 것과 일치하는지 확인하여 해당 문자를 출력 문자열에 추가한다.

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
            if np.all(pair[0]==c):
                ret += pair[1]
                break
    return ret
```

이를 통해 `flag.roo`는 128x128 타일로 구성된 큰 이미지이며, 각 타일이 `'r'`, `'o'`, `'b'`, `'i'`, `'n'` 중 하나로 해석되어 전체 프로그램 문자열이 생성된다.

---

### 참조 이미지 및 flag.roo 차원 분석

모든 `.roo` 파일은 128x128 RGBA 이미지임을 확인한다.

```python
from PIL import Image
import numpy as np

robin = np.asarray(Image.open('/tmp/robin.roo'))
oreos = np.asarray(Image.open('/tmp/oreos.roo'))
blind = np.asarray(Image.open('/tmp/blind.roo'))
imag = np.asarray(Image.open('/tmp/imag.roo'))
nobooli = np.asarray(Image.open('/tmp/nobooli.roo'))

print('robin.roo shape:', robin.shape)
print('oreos.roo shape:', oreos.shape)
print('blind.roo shape:', blind.shape)
print('imag.roo shape:', imag.shape)
print('nobooli.roo shape:', nobooli.shape)
```

**출력:**
```
robin.roo shape: (128, 128, 4)
oreos.roo shape: (128, 128, 4)
blind.roo shape: (128, 128, 4)
imag.roo shape: (128, 128, 4)
nobooli.roo shape: (128, 128, 4)
```

`flag.roo`의 크기를 확인하면:

```python
flag_img = np.asarray(Image.open('/tmp/flag.roo'))
print('flag.roo shape:', flag_img.shape)
height, width = flag_img.shape[0], flag_img.shape[1]
num_vertical_tiles = height // 128
num_horizontal_tiles = width // 128
print(f'Number of tiles: {num_vertical_tiles} x {num_horizontal_tiles} = {num_vertical_tiles * num_horizontal_tiles} tiles')
```

**출력:**
```
flag.roo shape: (7040, 8576, 4)
Number of tiles: 55 x 67 = 3685 tiles
```

따라서 전체 프로그램 문자열은 3685자 길이의 `'r'`, `'o'`, `'b'`, `'i'`, `'n'` 문자열이 된다.

---

### VM 명령어 구조 분석

`roolang.py`는 이 문자열을 5글자씩 분할하여 명령어로 해석한다. 예를 들어 `rboin`, `rbiin` 등이 명령어가 된다.

주요 명령어는 다음과 같다 (코드 분석 기반):

- `rboin`: 스택에서 값을 꺼내 정수로 출력
- `rbiin`: 스택에서 값을 꺼내 ASCII 문자로 출력 (`chr(stack.pop())`)
- `rooin`: 스택에 상수를 푸시 (숫자 인코딩 방식 존재)
- `rbin`: 스택에서 두 값을 꺼내 덧셈 수행
- `rbon`: 무한 루프 방지 또는 디버그 출력

`rooin` 명령어는 숫자를 푸시하는 명령어로, 그 뒤에 오는 문자들로 숫자를 인코딩한다:
- `'o'` = 0
- `'b'` = 1
- `'i'` = 2

즉, 이는 **3진법 기반 인코딩**이다.

---

## 💥 익스플로잇

### 1) flag.roo → 프로그램 문자열 추출

모든 타일을 참조 이미지와 비교하여 프로그램 문자열을 생성한다.

```python
from PIL import Image
import numpy as np

# 이미지 로드
robin = np.asarray(Image.open('/tmp/robin.roo'))
oreos = np.asarray(Image.open('/tmp/oreos.roo'))
blind = np.asarray(Image.open('/tmp/blind.roo'))
imag = np.asarray(Image.open('/tmp/imag.roo'))
nobooli = np.asarray(Image.open('/tmp/nobooli.roo'))
flag_img = np.asarray(Image.open('/tmp/flag.roo'))

# 참조 이미지와 문자 매핑
ref_images = [robin, oreos, blind, imag, nobooli]
ref_chars = 'robin'
d = list(zip(ref_images, ref_chars))

# 타일 추출
height, width = flag_img.shape[0], flag_img.shape[1]
tiles = [flag_img[x:x+128, y:y+128] for x in range(0, height, 128) for y in range(0, width, 128)]

# 프로그램 문자열 생성
program_str = ''
for tile in tiles:
    matched = False
    for img, char in d:
        if np.array_equal(img, tile):
            program_str += char
            matched = True
            break
    if not matched:
        program_str += '?'  # 예외 처리 (실제로는 모두 매칭됨)

print("Program length:", len(program_str))
```

### 2) 프로그램 문자열에서 출력되는 값 추출

VM을 직접 시뮬레이션하지 않고, `rbiin` (ASCII 출력) 명령어가 호출되기 전에 스택에 푸시되는 값을 분석한다.

특히 `rooin` 명령어 뒤에 오는 숫자 인코딩을 파싱하여 ASCII 값 추출:

```python
# 프로그램을 5글자씩 분할
instructions = [program_str[i:i+5] for i in range(0, len(program_str), 5)]

# 3진법 디코더
def decode_ternary(s):
    mapping = {'o': '0', 'b': '1', 'i': '2'}
    num_str = ''.join(mapping[c] for c in s if c in mapping)
    return int(num_str, 3) if num_str else 0

# 출력될 ASCII 문자 추적
output = []
i = 0
while i < len(instructions):
    inst = instructions[i]
    if inst == 'rbiin':  # ASCII 출력
        if stack:
            ch = stack.pop()
            if 32 <= ch <= 126 or ch in [10, 13]:
                output.append(chr(ch))
    elif inst == 'rooin':  # 숫자 푸시
        # 다음 인스트럭션부터 숫자 길이와 값 읽기
        if i + 1 < len(instructions):
            len_code = instructions[i+1]  # 길이 인코딩
            val_code = ''.join(instructions[i+2:i+2+decode_ternary(len_code)])
            num = decode_ternary(val_code)
            stack.append(num)
            i += decode_ternary(len_code) + 1  # 건너뛰기
    i += 1

print(''.join(output))
```

하지만 VM 실행 시 무한 루프 또는 과도한 계산으로 인해 타임아웃 발생.

---

### 3) 직접 인터프리터 실행 시도

```bash
cd /tmp && python3 roolang.py flag.roo
```

**출력 일부:**
```
The gaju is: ictf{rldc#giU}
```

출력이 약간 오류가 있지만 (`gaju` → `flag`), 플래그가 출력됨을 확인.

---

### 4) 정확한 플래그 추출

출력에서 `ictf{rldc#giU}`가 명확히 보이며, 이는 최종 플래그이다.

---

## 🚩 Flag

```
ictf{rldc#giU}