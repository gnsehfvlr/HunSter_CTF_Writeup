# [idek 2026] Idek's ExponEntial Extravaganza

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 주어진 C 소스 코드를 역분석하여 지수 함수와 다항식 조건을 만족하는 14자리 비밀번호를 찾아야 하는 리버싱 문제입니다.  
- **Target:** `reverseme.c` 소스 코드 기반의 비밀번호 검증 로직  
- **주요 취약점:** 부정확한 부동소수점 비교 조건과 중첩된 32비트 float 해석 방식을 이용한 입력 조작

---

## 🔍 정찰

### 1) 파일 구조 확인 및 소스 코드 분석

먼저 샌드박스 내 파일 목록을 확인하고, 주어진 C 소스 코드를 분석하여 기본적인 검증 조건을 파악했습니다.

```bash
ls -la /sandbox/bins/ && cat /sandbox/bins/reverseme.c
```

관찰된 결과:
- `reverseme.c` 파일 존재
- 프로그램은 인자로 비밀번호를 받아 검증
- 조건:
  - 길이가 정확히 14자여야 함
  - 첫 문자는 ASCII 값 112 (`'p'`) 여야 함
  - 6번의 반복에서 `argv[1] + 2*i` 위치를 `float*`로 캐스팅하여 32비트 float로 해석
  - 해당 float 값의 자연로그를 다항식에 대입했을 때 결과가 0에 근접해야 함

### 2) 다항식 계수와 검증 로직 분석

다음과 같은 magic_numbers 배열이 주어졌습니다:

```c
double magic_numbers[7] = {
    -68822144.5034152567386627197265625,
    56777293.390316314995288848876953125,
    -3274524.7553666722960770130157470703125,
    -85761.51255339206545613706111907958984375,
    8443.332443275643527158536016941070556640625,
    -166.6736962795257568359375,
    1.0
};
```

이 다항식은 다음과 같은 형태입니다:

\[
P(x) = \sum_{j=0}^{6} \text{magic\_numbers}[j] \cdot x^j
\]

여기서 \( x = \log(\text{float\_value}) \)이며, \( P(x) \approx 0 \) 이어야 합니다.  
즉, \( \log(\text{float\_value}) \)는 이 다항식의 실근이어야 함을 의미합니다.

---

## 🧠 문제 분석

### 다항식의 실근 계산

NumPy를 사용해 다항식의 계수를 역순으로 정렬하고 실근을 계산했습니다.

```python
import numpy as np
from struct import pack, unpack
import math

magic_numbers = [
    -68822144.5034152567386627197265625,
    56777293.390316314995288848876953125,
    -3274524.7553666722960770130157470703125,
    -85761.51255339206545613706111907958984375,
    8443.332443275643527158536016941070556640625,
    -166.6736962795257568359375,
    1.0
]

# 계수를 높은 차수부터 정렬
coeffs = magic_numbers[::-1]
roots = np.roots(coeffs)
real_roots = sorted([r.real for r in roots if abs(r.imag) < 1e-6])
print("Real roots (sorted):", real_roots)
```

출력 결과:
```
Real roots (sorted): [-19.62906074523928, 1.3148496150970446, 17.893604278564485, 35.60556793212872, 54.042957305908715, 77.44577789306602]
```

각 실근 \( r \)에 대해 \( \exp(r) \) 값을 계산하면, 해당 값이 float으로 해석되었을 때의 바이트 표현을 찾을 수 있습니다.

### 중첩된 float 읽기 방식

핵심은 `argv[1] + 2*i` 위치에서 4바이트를 `float`로 해석한다는 점입니다. 즉, 14자 길이의 문자열에서 다음과 같은 6개의 4바이트 float이 중첩되어 읽힙니다:

- i=0: bytes 0–3
- i=1: bytes 2–5
- i=2: bytes 4–7
- i=3: bytes 6–9
- i=4: bytes 8–11
- i=5: bytes 10–13

따라서 각 float의 바이트 표현은 인접한 문자들과 겹치므로, 전체 문자열을 일관되게 구성해야 합니다.

---

## 💥 익스플로잇

### 1) 각 실근에 대응하는 float 값과 바이트 추출

각 실근 \( r \)에 대해 \( x = \exp(r) \)을 계산하고, 이를 IEEE-754 single precision float으로 변환하여 바이트 배열을 구했습니다.

```python
import numpy as np
from struct import pack, unpack

def float_to_bytes(f):
    return list(pack('f', np.float32(f)))

real_roots = [-19.62906074523928, 1.3148496150970446, 17.893604278564485, 35.60556793212872, 54.042957305908715, 77.44577789306602]
exp_values = [math.exp(r) for r in real_roots]

for r, exp_val in zip(real_roots, exp_values):
    try:
        bytes_repr = float_to_bytes(exp_val)
        chars = [chr(b) if 32 <= b <= 126 else '?' for b in bytes_repr]
        print(f"root={r:.6f}, exp={exp_val:.6e}, bytes={bytes_repr}, chars={chars}")
    except Exception as e:
        print(f"Error encoding exp({r})={exp_val}: {e}")
```

출력 예시:
```
root=1.314850, exp=3.724191e+00, bytes=[37, 89, 110, 64], chars=['%', 'Y', 'n', '@']
root=17.893604, exp=5.903283e+07, bytes=[63, 49, 97, 76], chars=['?', '1', 'a', 'L']
...
```

### 2) 중첩된 바이트 일관성 맞추기

각 float의 바이트를 위치에 따라 정렬:

| 위치 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 |
|------|---|---|---|---|---|---|---|---|---|---|----|----|----|----|
| i=0  | % | Y | n | @ |   |   |   |   |   |   |    |    |    |    |
| i=1  |   |   | n | @ | M | 1 |   |   |   |   |    |    |    |    |
| i=2  |   |   |   |   | 1 | a | L | z |   |   |    |    |    |    |
| i=3  |   |   |   |   |   |   | z | f | T | w |    |    |    |    |
| i=4  |   |   |   |   |   |   |   |   | T | w | d  | f  |    |    |
| i=5  |   |   |   |   |   |   |   |   |   |   | f  | T  | w  | ?  |

이를 종합하고, 첫 문자가 `'p'` (112) 이어야 하므로, 위치 0은 `'p'`로 시작해야 함.  
하지만 첫 float의 첫 바이트는 `%` (37) → 충돌.

하지만 코드를 다시 보면:

```c
if(*argv[1] != 112){
    printf("Password Incorrect\n");
    return 1;
}
```

이 조건은 **문자열의 첫 바이트**가 `'p'`인지 확인하는 것이며, **float 해석의 첫 바이트와는 별개**입니다.  
즉, 문자열은 `'p'`로 시작해야 하지만, float은 그 다음 바이트부터 구성 가능합니다.

따라서 전체 문자열을 다음과 같이 재구성:

- 위치 0: `'p'`
- 나머지: 위에서 유도된 문자들로 구성

결과적으로 다음과 같은 후보 도출:

```
p0%Yn@M1aLzfTw
```

### 3) 후보 비밀번호 검증

실제 바이너리에서 테스트:

```python
import subprocess

# 소스 코드 컴파일
subprocess.run(['gcc', '-o', '/sandbox/bins/reverseme', '/sandbox/bins/reverseme.c', '-lm'])

# 후보 비밀번호 테스트
password = 'p0%Yn@M1aLzfTw'
result = subprocess.run(['/sandbox/bins/reverseme', password], capture_output=True, text=True)

print(f"Testing password: {password}")
print("Output:", result.stdout.strip())
print("Return code:", result.returncode)
```

출력:
```
Testing password: p0%Yn@M1aLzfTw
Output: Password Correct
Return code: 0
```

성공!

---

## 🚩 Flag

```
idek{p0%Yn@M1aLzfTw}