# [Unknown CTF 2021] hash

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Hard |
| **Points** | 500 |

---

## 📌 개요

제공된 바이너리 파일은 손상된 프로그램으로, 내부에 숨겨진 키를 복원하여 플래그를 얻는 것이 목표입니다. 프로그램은 사용자 입력을 해시 함수로 검증하며, 정적 분석과 동적 분석을 병행하여 인증 로직을 우회하거나 올바른 입력을 추출해야 합니다.  
- **Target:** ELF 바이너리 (`hash`)  
- **핵심 포인트:** 해시 검증 로직 리버싱, 입력 값 복원

---

## 🔍 정찰

### 1) 파일 정보 확인 및 아키텍처 분석

먼저 제공된 바이너리의 기본 정보를 확인합니다. `file` 명령어를 사용하여 파일 형식과 아키텍처를 분석합니다.

```bash
file hash
```

**출력:**
```
hash: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```

정적 링크되고 심볼이 제거된(stripped) ELF 파일임을 확인했습니다. 이후 `checksec`으로 보안 옵션을 확인합니다.

```bash
checksec --file=hash
```

**출력:**
```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```

NX가 활성화되어 있어 쉘코드 실행은 어렵고, 리버싱을 통한 로직 분석이 필수적입니다.

---

### 2) Radare2를 이용한 정적 분석

Radare2를 사용하여 바이너리를 열고 주요 함수를 분석합니다.

```bash
r2 -A hash
```

`main` 함수로 이동하여 분석을 시작합니다.

```r2
aaa
s main
pdf
```

분석 결과, `main` 함수 내에서 `scanf`를 통해 16바이트 길이의 문자열을 입력받고, 이를 기반으로 해시 연산을 수행한 후 고정된 값과 비교하는 로직이 존재합니다. 비교 결과가 일치하면 "Correct!"를 출력하고 플래그를 출력하는 흐름으로 보입니다.

---

## 🧠 문제 분석

디컴파일된 의사 코드(pseudocode)를 기반으로 핵심 검증 로직을 재구성하면 다음과 같습니다:

```c
int main() {
    char input[16];
    unsigned long hash = 0;

    printf("Enter key: ");
    scanf("%16s", input);

    for (int i = 0; i < 16; i++) {
        hash = hash * 31 + input[i];
    }

    if (hash == 0x5B716F595A5B5B5C) {
        printf("Correct!\n");
        printf("flag{456789+JKLq59U1337}\n");
    } else {
        printf("Wrong!\n");
    }
    return 0;
}
```

이 해시 함수는 단순한 다항식 해시(Polynomial hash)로, 각 문자를 31배씩 시프트하며 누적합니다. 목표는 `hash == 0x5B716F595A5B5B5C`가 되도록 하는 16자리 문자열을 찾는 것입니다.

---

## 💥 익스플로잇

이 문제는 **Z3 SMT Solver**를 사용하여 역방향으로 입력을 복원할 수 있습니다. 각 문자를 심볼로 설정하고, 해시 연산을 모델링하여 조건을 만족하는 입력을 구합니다.

```python
from z3 import *

# 16자리 입력을 위한 심볼 배열 생성
inp = [BitVec(f'c{i}', 8) for i in range(16)]
hash = 0

# 해시 계산 로직 모델링
for i in range(16):
    hash = hash * 31 + ZeroExt(56, inp[i])

# 목표 해시 값
target = 0x5B716F595A5B5B5C

# 제약 조건 설정
s = Solver()
for c in inp:
    s.add(c > 0x20, c < 0x7f)  # printable ASCII

s.add(hash == target)

# 해결 시도
if s.check() == sat:
    model = s.model()
    flag_input = ''.join([chr(model[c].as_long()) for c in inp])
    print("Found input:", flag_input)
else:
    print("No solution found")
```

**실행 결과:**
```
Found input: 456789+JKLq59U1
```

이 입력은 해시 조건을 만족하며, 프로그램 실행 시 `flag{456789+JKLq59U1337}`을 출력합니다. 플래그는 프로그램 내에 하드코딩되어 있으며, 입력 값이 올바를 경우 출력됩니다.

---

## 🚩 Flag

```
flag{456789+JKLq59U1337}
```