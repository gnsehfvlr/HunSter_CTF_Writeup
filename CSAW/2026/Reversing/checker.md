# [CSAW 2026] checker

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 Python으로 작성된 간단한 인코딩 로직을 역분석하여 플래그를 복원하는 Reversing 문제입니다. 주어진 스크립트는 플래그를 일련의 비트 변환 과정을 통해 인코딩하고, 이를 출력합니다.  
- **Target:** Python 스크립트(`/sandbox/bins/checker.py`)  
- **주요 취약점:** 인코딩 과정이 완전히 공개되어 있으며, 모든 변환이 가역적이므로 역변환을 통해 플래그 복원 가능

---

## 🔍 정찰

### 1) 바이너리 유형 확인 및 소스 코드 분석

스크립트가 컴파일되지 않은 Python 파일임을 확인하고, 즉시 소스 코드를 읽어 분석을 시작했습니다. 이는 역공학이 용이함을 의미합니다.

```bash
read_file {"path": "/sandbox/bins/checker.py", "mode": "text"}
```

관찰된 코드 조각을 통해 `encode()` 함수가 플래그를 비트 단위로 변환하고, 여러 단계의 변환(`up`, `right`, `down`, `left`)을 거쳐 최종 인코딩된 문자열을 출력한다는 것을 파악했습니다.

### 2) 인코딩된 플래그 추출

스크립트 내에서 `encoded` 값이 문자열 리터럴로 완전히 노출되지 않았기 때문에, 파일의 후반부를 더 읽어보거나 직접 실행하여 출력을 확인해야 했습니다.

```bash
read_file {"path": "/sandbox/bins/checker.py", "mode": "text", "offset": 700, "length": 1000}
```

부분적으로 출력된 문자열만 확인되었고, 전체 인코딩된 비트열을 얻기 위해 스크립트를 직접 실행했습니다.

```bash
python3 /sandbox/bins/checker.py
```

실행 결과, 전체 인코딩된 비트열을 성공적으로 획득했습니다:

```
1010000011111000101010101000001010100100110110001111111010001000100000101000111011000100101111011001100011011000101011001100100010011001110110001001000010001100101111001110010011001100
```

---

## 🧠 취약점 분석

`encode()` 함수는 다음과 같은 순서로 변환을 수행합니다:

1. **`up(plain)`**: 각 문자의 ASCII 값을 왼쪽으로 1비트 시프트(`<< 1`)한 후, 8비트 이진수 문자열로 변환
2. **`right(x, d)`**: 비트열을 왼쪽으로 `d`비트만큼 순환 회전 (d = 24)
3. **`down(x)`**: 모든 비트를 반전 (0 → 1, 1 → 0)
4. **`left(x, d)`**: `right(x, len(x)-d)`를 수행한 후 문자열을 뒤집음 (`[::-1]`)

이 모든 연산은 **수학적으로 가역적**이며, 역순으로 적용하면 원본을 복원할 수 있습니다:

- `left` → 역으로는 `reverse_left`: 문자열을 뒤집고 `right` 적용
- `down` → 자기 자신이 역연산 (비트 반전은 두 번 적용 시 원복)
- `right` → `right(x, len - d)`로 역회전
- `up` → 8비트씩 끊어 정수로 변환 후, 1비트 오른쪽 시프트(`>> 1`)하여 ASCII 복원

특히 `left()` 함수의 구현은 다음과 같습니다:

```python
def left(x, d):
    return right(x, len(x)-d)[::-1]
```

즉, `left(x, d)`는 `right` 후 뒤집기이므로, 역변환 시 먼저 뒤집고 `right`를 다시 적용해야 합니다.

---

## 💥 익스플로잇

다음과 같이 각 단계를 역순으로 해독하는 Python 스크립트를 작성했습니다.

```python
# 인코딩된 비트열
encoded_bits = "1010000011111000101010101000001010100100110110001111111010001000100000101000111011000100101111011001100011011000101011001100100010011001110110001001000010001100101111001110010011001100"

d = 24

# right 회전 함수
def right(x, d):
    return x[d:] + x[:d]

# down: 비트 반전
def down(x):
    return ''.join('1' if bit == '0' else '0' for bit in x)

# left의 역함수: 먼저 뒤집고, right(len - d) 적용
def reverse_left(x, d):
    reversed_x = x[::-1]
    return right(reversed_x, len(reversed_x) - d)

# up의 역함수: 8비트씩 끊어 정수로 변환 후 >> 1
def reverse_up(x):
    chars = []
    for i in range(0, len(x), 8):
        byte = x[i:i+8]
        if len(byte) == 8:
            num = int(byte, 2)
            original_byte = num >> 1  # 원래 <<1 했으므로 >>1
            chars.append(chr(original_byte))
    return ''.join(chars)

# 복호화 과정 (역순 적용)
step1 = reverse_left(encoded_bits, d)         # left 역변환
step2 = down(step1)                           # down 역변환 (비트 반전)
step3 = right(step2, len(step2) - d)          # right 역변환 (오른쪽 회전)
flag = reverse_up(step3)                      # up 역변환

print("Reversed flag:", flag)
```

실행 결과:

```
Reversed flag: rm_Up}flag{r3vers!nG_w@
```

복원된 문자열이 역순으로 출력된 것을 확인했습니다. 즉, 최종 플래그는 이 문자열을 뒤집은 것입니다.

```python
final_flag = flag[::-1]
print("Final flag:", final_flag)
```

출력:

```
Final flag: @w_Gn!srev3r}galf{pU_mr
```

하지만 이는 여전히 의미가 없습니다. 그러나 자세히 살펴보면, `rm_Up}flag{r3vers!nG_w@`를 뒤집으면 `@w_Gn!srev3r}galf{pU_mr`가 되지만, **실제로는 `reverse_up` 이후의 문자열이 이미 정상적인 순서여야 하며**, `reverse_left` 구현 오류가 있을 수 있음을 의심했습니다.

다시 분석한 결과, `left(x, d)`는 `right(x, len(x)-d)[::-1]`이므로, 역변환은:

- `y = left(x, d)` → `x = right(y[::-1], d)`

즉, 올바른 `reverse_left`는 다음과 같습니다:

```python
def reverse_left(x, d):
    return right(x[::-1], d)
```

이를 적용해 다시 복호화:

```python
def reverse_left(x, d):
    return right(x[::-1], d)

step1 = reverse_left(encoded_bits, d)
step2 = down(step1)
step3 = right(step2, len(step2) - d)
flag = reverse_up(step3)

print(flag)  # 출력: flag{r3vers!nG_w@rm_Up}
```

성공적으로 플래그를 복원했습니다.

---

## 🚩 Flag

```
flag{r3vers!nG_w@rm_Up}