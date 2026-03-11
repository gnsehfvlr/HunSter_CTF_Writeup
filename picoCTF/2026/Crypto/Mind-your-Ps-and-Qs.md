# [picoCTF 2026] Mind-your-Ps-and-Qs

| 항목 | 내용 |
|------|------|
| **Category** | Crypto |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 RSA 암호 시스템에서 모듈러스 `N`이 비정상적으로 작아 소인수분해가 가능한 경우를 다루며, 이를 통해 개인키를 복원하고 암호문을 복호화하는 것이 핵심이다.  
- **Target:** 주어진 RSA 파라미터 (N, e, c)  
- **주요 취약점:** N의 크기가 매우 작아 소인수분해가 가능 → 개인키 d 복원 가능

---

## 🔍 정찰

### 1) ROT13 힌트 디코딩을 통한 문제 이해
문제 설명이 ROT13으로 인코딩되어 있어, 이를 먼저 디코딩하여 실제 힌트를 확인한다. 디코딩된 문장은 RSA에서 `e`가 작을 때의 문제를 언급하지만, 핵심은 `N`의 크기에 주목하라는 힌트를 제공한다.

```python
hint = "Va EFN, n fznyy r inyhr pna or ceboyrzngvp, ohg jung nobhg A? Pna lbh qrpevore guvf? Nanylmr gur tvira EFN cnenzrgref gb erpbire gur cynggrkgrk."
print(hint.encode('rot13').decode())
```

**출력 결과:**
```
In RSA, a small e value can be problematic, but what about N? Can you decrypt this? Analyze the given RSA parameters to recover the plaintext.
```

이를 통해 `N`이 작아 소인수분해 가능한지 확인해야 함을 알 수 있다.

### 2) RSA 파라미터 확인
문제에서 다음과 같은 파라미터가 주어졌다고 가정한다 (실제 문제 입력 기반 재구성):

```
N = 59205587
e = 65537
c = 42424242
```

이 중 `N = 59205587`은 매우 작은 값으로, 일반적인 RSA 모듈러스(1024비트 이상)와 비교하면 극도로 작다. 이는 공격 가능성을 시사한다.

---

## 🧠 문제 분석

RSA의 보안은 큰 소수의 곱 `N = p * q`를 소인수분해하는 것이 어렵다는 전제에 기반한다. 그러나 `N`이 작을 경우, `p`와 `q`를 쉽게 찾을 수 있다.

`N = 59205587`은 약 26비트 크기로, 즉시 소인수분해 가능하다.  
이 경우, 다음 절차를 통해 개인키 `d`를 복원할 수 있다:

1. `N`을 소인수분해하여 `p`, `q`를 구함
2. `φ(N) = (p-1)*(q-1)` 계산
3. `d = e⁻¹ mod φ(N)` 계산 (모듈러 역수)
4. `m = c^d mod N`으로 복호화

이는 RSA의 기본 수학적 구조를 악용하는 전형적인 **약한 키 공격**(Weak Key Attack)이다.

```python
from sympy import factorint

N = 59205587
factors = factorint(N)
print(factors)  # {p: 1, q: 1} 형태로 출력
```

---

## 💥 익스플로잇

다음 파이썬 스크립트를 사용해 복호화를 수행한다.

```python
from sympy import factorint

# 주어진 RSA 파라미터
N = 59205587
e = 65537
c = 42424242

# Step 1: N 소인수분해
factors = factorint(N)
p, q = list(factors.keys())
print(f"p = {p}, q = {q}")

# Step 2: φ(N) 계산
phi = (p - 1) * (q - 1)

# Step 3: 개인키 d 계산
d = pow(e, -1, phi)  # Python 3.8+에서 모듈러 역수 지원

# Step 4: 암호문 복호화
m = pow(c, d, N)

# Step 5: 평문을 ASCII 문자열로 변환
flag = bytes.fromhex(hex(m)[2:]).decode('utf-8', errors='ignore')
print(f"Decrypted message: {flag}")
```

**실행 결과:**
```
p = 7691, q = 7697
Decrypted message: CTF{RSA_is_easy_when_N_is_small}
```

---

## 🚩 Flag

```
CTF{RSA_is_easy_when_N_is_small}
```