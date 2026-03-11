# [picoCTF 2026] crackme-py

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 주어진 Python 스크립트를 역분석하여 숨겨진 플래그를 추출하는 Reversing 문제입니다.  
스크립트 내에 암호화된 문자열과 ROT47 복호화 함수가 포함되어 있으나, 직접 실행 시 플래그가 출력되지 않으며, 수동으로 복호화 로직을 분석하고 구현해야 합니다.  
- **Target:** `crackme.py` Python 스크립트  
- **주요 취약점:** 암호화된 플래그가 소스코드에 하드코딩되어 있으며, 간단한 ROT47 인코딩을 사용해 보호됨

---

## 🔍 정찰

### 1) 파일 목록 확인 및 스크립트 내용 분석

먼저, 제공된 샌드박스 디렉터리 내 파일을 확인하여 분석 대상을 파악합니다.

```bash
ls -la /sandbox/bins/
```

출력 결과, 유일한 파일인 `crackme.py` 가 존재함을 확인합니다.  
이제 해당 파일의 내용을 확인합니다.

```bash
cat /sandbox/bins/crackme.py
```

스크립트 내용을 살펴보면 다음과 같은 핵심 요소가 포함되어 있습니다:

- `bezos_cc_secret = "A:4@r%uL`M-^M0c0AbcM-MFE055a4ce`eN"` — 암호화된 비밀 값
- `alphabet`: ASCII 33~126 범위의 프린터블 문자로 구성된 참조 알파벳
- `decode_secret(secret)`: ROT47 방식으로 문자를 디코딩하는 함수
- `choose_greatest()`: 사용자 입력을 받아 두 수 중 큰 값을 출력하는 함수 (플래그와 무관)

### 2) 스크립트 동작 확인

스크립트를 직접 실행하여 동작을 확인합니다.

```bash
python3 /sandbox/bins/crackme.py
```

실행 결과, 사용자에게 두 개의 숫자를 입력받고 그 중 큰 값을 출력하는 단순한 동작만 수행하며, `decode_secret` 함수는 호출되지 않습니다.  
이는 플래그가 **자동으로 출력되지 않으며**, 수동으로 `decode_secret(bezos_cc_secret)` 를 호출해야 함을 의미합니다.

---

## 🧠 문제 분석

`decode_secret` 함수는 ROT47 알고리즘을 구현하고 있습니다.  
ROT47은 카이사르 암호의 일종으로, 프린터블 ASCII 문자(33~126, 총 94문자)를 기준으로 47칸씩 시프트하는 인코딩 방식입니다.  
인코딩과 디코딩이 동일한 연산이므로, 동일한 함수를 두 번 적용하면 원본이 복구됩니다.

함수 내부 로직은 다음과 같습니다:

```python
def decode_secret(secret):
    rotate_const = 47
    decoded = ""
    for c in secret:
        index = alphabet.find(c)
        original_index = (index + rotate_const) % len(alphabet)
        decoded = decoded + alphabet[original_index]
    return decoded
```

여기서 중요한 점은:
- `alphabet` 문자열이 정확히 94개의 문자로 구성되어야 함 (ASCII 33~126)
- `alphabet.find(c)` 가 -1을 반환하면 문자가 알파벳에 없음을 의미하므로 예외 처리 필요
- 원래의 `alphabet` 정의가 줄바꿈으로 인해 파이썬 문법 오류를 일으킬 수 있음

이전 시도에서 `alphabet` 을 잘못 정의했기 때문에, `A` 가 `p` 로 매핑되지 않고 `!` 로만 구성된 문자열이 출력되었습니다.  
이는 `alphabet` 이 제대로 구성되지 않아 모든 문자의 `index` 가 -1이거나 잘못 계산되었기 때문입니다.

---

## 💥 익스플로잇

정확한 `alphabet` 을 생성하고, `decode_secret` 함수를 올바르게 구현한 후 `bezos_cc_secret` 을 복호화합니다.

```python
# Correctly define the alphabet from ! to ~ (ASCII 33 to 126)
alphabet = ''.join([chr(i) for i in range(33, 127)])

def decode_secret(secret):
    rotate_const = 47
    decoded = ""
    for c in secret:
        index = alphabet.find(c)
        if index == -1:
            decoded += c  # keep characters not in alphabet
        else:
            original_index = (index + rotate_const) % len(alphabet)
            decoded += alphabet[original_index]
    return decoded

# Decrypt the secret
bezos_cc_secret = "A:4@r%uL`M-^M0c0AbcM-MFE055a4ce`eN"
flag = decode_secret(bezos_cc_secret)
print(flag)
```

실행 결과:

```
picoCTF{1|\/|_4_p34|\|ut_dd2c4616}
```

이 플래그는 ROT47로 인코딩된 `bezos_cc_secret` 을 정확한 알파벳 범위로 복호화했을 때 얻어지며, 문제의 정답입니다.

---

## 🚩 Flag

```
picoCTF{1|\/|_4_p34|\|ut_dd2c4616}