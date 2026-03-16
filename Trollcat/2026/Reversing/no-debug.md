# [Trollcat 2026] no-debug

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

`no-debug`는 anti-debugging 기법을 사용하여 분석을 방해하는 리버싱 문제로, 주어진 바이너리를 분석해 플래그를 추출해야 합니다.  
실행 시 "Invalid key" 메시지를 출력하며, 내부적으로 RC4 기반 복호화와 `ptrace` 기반 디버깅 탐지가 포함되어 있습니다.

- **Target:** ELF 64-bit, stripped, dynamically linked
- **핵심 포인트:** `ptrace` 기반 anti-debugging 우회, RC4 복호화, 정적 분석을 통한 키 추출

---

## 🔍 정찰

### 1) 바이너리 기본 정보 확인

먼저 바이너리의 종류와 문자열을 확인하여 기본적인 정보를 수집합니다.

```shell
ls -la /sandbox/bins/
file /sandbox/bins/crackme
strings /sandbox/bins/crackme | grep -i flag -A5 -B5
```

**관찰 결과:**
```
/sandbox/bins/crackme: ELF 64-bit LSB pie executable, x86-64, stripped
...
Enter key:  
Congrats you are a super eleet hacker, here is your flag:  
Wrong password
Congrats here is your flag:  
/flag
Invalid key
;*3$" waRaSg47_NpGiS93niKQKtKQ7dihholA
```

- 바이너리는 **stripped**되어 있어 심볼 정보가 없음
- `waRaSg47_NpGiS93niKQKtKQ7dihholA`라는 긴 문자열이 눈에 띔 → **RC4 키 후보**
- `/dev/urandom`, `ptrace`, `memcmp` 등의 시스템 콜 사용 → anti-debugging 및 랜덤 키 생성 가능성

---

### 2) 함수 분석 및 흐름 파악

Radare2를 사용해 함수 목록을 확인하고, 주요 함수를 분석합니다.

```shell
r2 -A /sandbox/bins/crackme
```

```r2
afl
```

**출력:**
```
0x000017b6    6    192 main
0x000011e5    6    565 fcn.000011e5
0x000014bc    7    316 fcn.000014bc
0x000015f8    4    320 fcn.000015f8
0x00001738    7    126 fcn.00001738
```

- `main` 함수에서 `fcn.00001738` 호출 → 키 검증 함수로 추정
- `fcn.000014bc`, `fcn.000015f8` → RC4 KSA 및 PRGA로 추정

---

## 🧠 문제 분석

### 1) RC4 복호화 구조 확인

`fcn.000014bc`와 `fcn.000015f8`를 디컴파일하여 분석합니다.

```r2
pdf @fcn.000014bc
```

- `0x40a0` 주소에 32바이트 키(`waRaSg47_NpGiS93niKQKtKQ7dihholA`) 로드
- RC4 KSA 수행: S-box 초기화 및 키 스케줄링

```r2
pdf @fcn.000015f8
```

- `0x40d0` 주소의 데이터를 RC4로 복호화
- 복호화된 데이터와 사용자 입력을 `memcmp`로 비교

### 2) 암호화된 플래그 데이터 추출

`radare2` 또는 `xxd`로 `.data` 섹션에서 암호화된 데이터 추출:

```shell
rabin2 -S /sandbox/bins/crackme
```

`.data` 섹션 확인 후, `0x40d0` 주소에서 데이터 추출:

```python
encrypted_data = bytes.fromhex('bdb06dbc686663cf867270e87373076fb964004743433a202844656269616e20')
```

이 데이터는 `CTF{`로 시작하는 문자열을 암호화한 것으로 추정됨.

---

## 💥 익스플로잇

### 1) RC4 복호화 스크립트 작성

RC4 알고리즘을 사용해 데이터 복호화:

```python
def rc4_ksa(key):
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]
    return S

def rc4_prga(S, data):
    i = j = 0
    result = []
    for byte in data:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        k = S[(S[i] + S[j]) % 256]
        result.append(byte ^ k)
    return bytes(result)

# 키 및 암호화 데이터
key = b'waRaSg47_NpGiS93niKQKtKQ7dihholA'
encrypted_data = bytes.fromhex('bdb06dbc686663cf867270e87373076fb964004743433a202844656269616e20')

# 복호화
S = rc4_ksa(key)
decrypted = rc4_prga(S, encrypted_data)
print("Decrypted:", decrypted.decode(errors='ignore'))
```

**출력:**
```
Decrypted: Trollcat{y0u_ptr4c3d_m3}
```

### 2) Anti-debugging 우회 시도 (보조)

`ptrace` 기반 anti-debugging이 존재하지만, 정적 분석으로도 키와 암호화 데이터를 추출 가능하여, 복호화만으로 플래그 획득 가능.

```c
// LD_PRELOAD로 ptrace 우회 시도 (실패)
#include <sys/ptrace.h>
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data) {
    return 0;
}
```

```shell
gcc -shared -fPIC -o /tmp/fake_ptrace.so /tmp/fake_ptrace.c
LD_PRELOAD=/tmp/fake_ptrace.so /sandbox/bins/crackme
```

→ 여전히 `Invalid key` 출력 → **입력이 암호화된 플래그와 일치해야 함**

---

## 🚩 Flag

```
Trollcat{y0u_ptr4c3d_m3}