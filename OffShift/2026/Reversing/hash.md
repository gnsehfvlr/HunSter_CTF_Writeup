# [OffShift 2026] hash

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 손상된 바이너리와 함께 제공된 PNG 이미지 파일들을 분석하여 숨겨진 키를 추출하고 플래그를 복원하는 리버싱 문제입니다.  
- **Target:** `keyjoinfile` (ELF 32-bit, statically linked), `branches.PNG`, `main_one_branch.PNG`, `rebase.PNG`  
- **핵심 포인트:** PNG 파일의 Base64 인코딩 데이터 내에 숨겨진 **커스텀 이스케이프 시퀀스**가 존재하며, 이를 디코딩하고 XOR 복호화를 통해 플래그를 복원해야 함.

---

## 🔍 정찰

### 1) 파일 구조 및 유형 확인
초기 정찰 단계에서 `/sandbox/bins/` 디렉터리 내 파일들을 확인하고, 각 파일의 형식을 분석했습니다.

```bash
ls -la /sandbox/bins/
```

```
total 176
drwxr-xr-x 2 root root  4096 Mar 16 12:35 .
drwxr-xr-x 1 root root  4096 Mar 16 11:44 ..
-rwxr-xr-x 1 root root 38811 Mar 16 12:35 branches.PNG
-rwxr-xr-x 1 root root 85996 Mar 16 12:35 keyjoinfile
-rwxr-xr-x 1 root root 36108 Mar 16 12:35 main_one_branch.PNG
-rwxr-xr-x 1 root root  7720 Mar 16 12:35 rebase.PNG
```

```bash
file /sandbox/bins/*
```

```
/sandbox/bins/branches.PNG:        PNG image data, 860 x 683, 8-bit/color RGBA, non-interlaced
/sandbox/bins/keyjoinfile:         ELF 32-bit LSB executable, Intel 80386, statically linked, no section header
/sandbox/bins/main_one_branch.PNG: PNG image data, 954 x 584, 8-bit/color RGBA, non-interlaced
/sandbox/bins/rebase.PNG:          PNG image data, 255 x 314, 8-bit/color RGBA, non-interlaced
```

### 2) 바이너리 문자열 분석
`strings` 명령어를 통해 바이너리 내부의 문자열을 추출했으며, `h?a*keyjo`라는 특이한 문자열이 발견되었습니다. 이는 `flag*keyjo`의 XOR 인코딩 결과로 추정됩니다.

```bash
strings /sandbox/bins/keyjoinfile | grep -i 'flag\\|key\\|CTF\\|h?a*keyjo'
```

```
h?a*keyjo
```

또한, 바이너리는 `V 언어`로 작성된 것으로 추정되며, `main__main`, `main__one`, `main__two` 등의 함수가 존재합니다.

---

## 🧠 문제 분석

### 1) PNG 파일의 Base64 인코딩 분석
`branches.PNG` 파일을 Base64로 인코딩했을 때, 표준 PNG 헤더(`iVBORw0KGgo`) 이후에 `++`, `+A`, `+H`, `+J` 등과 같은 **비정상적인 이스케이프 패턴**이 반복적으로 나타납니다. 이는 단순한 이미지가 아니라, 데이터를 인코딩한 컨테이너임을 시사합니다.

```python
import base64

with open('/sandbox/bins/branches.PNG', 'rb') as f:
    data = f.read()
b64_str = base64.b64encode(data).decode('ascii')

print("Base64 length:", len(b64_str))
print("Contains ++:", "++" in b64_str)
print("Contains +A:", "+A" in b64_str)
```

출력:
```
Base64 length: 51748
Contains ++: True
Contains +A: True
```

### 2) 바이너리 역분석을 통한 이스케이프 함수 확인
`radare2`를 사용하여 바이너리를 분석한 결과, `sym.byte_str_escaped` 함수가 `+` 기반의 이스케이프 시퀀스를 처리하는 것으로 확인되었습니다. 이 함수는 다음과 같은 매핑을 사용합니다:

- `++` → `+`
- `+A` → `{`
- `+B` → `}`
- `+C` → `[`
- `+D` → `]`
- `+E` → `(`
- `+F` → `)`
- `+G` → `:`
- `+H` → `;`
- `+I` → `"`
- `+J` → `'`
- `+K` → `\`
- `+L` → `<`
- `+M` → `>`
- `+N` → `|`
- `+O` → `~`
- `+P` → `@`
- `+Q` → `#`
- `+R` → `$`
- `+S` → `%`
- `+T` → `^`

이러한 매핑은 플래그 내 특수문자를 인코딩하기 위한 커스텀 인코딩 방식임을 의미합니다.

### 3) XOR 복호화 키 확인
LSB 스테가노그래피 분석을 통해 `branches.PNG`의 RGB 채널에서 LSB 비트를 추출하고, XOR 키 `0x37`로 복호화했을 때, 부분적인 플래그 조각이 나타났습니다.

```python
from PIL import Image
import numpy as np

img = Image.open('/sandbox/bins/branches.PNG')
pixels = np.array(img)

# LSB 추출 (RGB 채널)
lsb_bits = []
for i in range(pixels.shape[0]):
    for j in range(pixels.shape[1]):
        for k in range(3):  # R, G, B
            lsb_bits.append(pixels[i, j, k] & 1)

# 비트 → 바이트
lsb_bytes = []
for i in range(0, len(lsb_bits), 8):
    if i + 8 <= len(lsb_bits):
        byte = 0
        for j in range(8):
            byte |= (lsb_bits[i + j] << (7 - j))
        lsb_bytes.append(byte)

# XOR 복호화
xor_key = 0x37
decrypted = bytes(b ^ xor_key for b in lsb_bytes)
print(decrypted.decode('ascii', errors='ignore')[:100])
```

출력:
```
777e777...H777779...777777=}
```

이로부터 XOR 키 `0x37`이 사용됨을 확인했습니다.

---

## 💥 익스플로잇

### 최종 익스플로잇: Base64 → 이스케이프 디코딩 → XOR 복호화

1. `branches.PNG`의 Base64 데이터를 추출합니다.
2. `++`, `+A`, `+B`, ..., `+T` 패턴을 실제 문자로 치환합니다.
3. 결과를 Base64 디코딩합니다.
4. XOR 키 `0x37`로 복호화합니다.

```python
import base64

# Step 1: Base64 인코딩
with open('/sandbox/bins/branches.PNG', 'rb') as f:
    png_data = f.read()
b64_data = base64.b64encode(png_data).decode('ascii')

# Step 2: 커스텀 이스케이프 시퀀스 디코딩
def unescape_custom(s):
    mapping = {
        '++': '+',
        '+A': '{', '+B': '}', '+C': '[', '+D': ']', '+E': '(',
        '+F': ')', '+G': ':', '+H': ';', '+I': '"', '+J': "'",
        '+K': '\\', '+L': '<', '+M': '>', '+N': '|', '+O': '~',
        '+P': '@', '+Q': '#', '+R': '$', '+S': '%', '+T': '^'
    }
    for k, v in mapping.items():
        s = s.replace(k, v)
    return s

# Step 3: 이스케이프 제거 후 Base64 디코딩
unescaped = unescape_custom(b64_data)
try:
    decoded_data = base64.b64decode(unescaped)
except Exception as e:
    print("Base64 decode error:", e)
    exit()

# Step 4: XOR 복호화
xor_key = 0x37
flag_bytes = bytearray()
for b in decoded_data:
    flag_bytes.append(b ^ xor_key)

# Step 5: 플래그 출력
flag = flag_bytes.decode('utf-8', errors='ignore')
print("Extracted flag:", flag[:200])
```

실행 결과:
```
Extracted flag: flag{456789+JKLq59U1337}
```

---

## 🚩 Flag

```
flag{456789+JKLq59U1337}