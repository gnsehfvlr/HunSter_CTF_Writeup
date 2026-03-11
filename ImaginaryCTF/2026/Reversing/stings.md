# [ImaginaryCTF 2026] stings

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 ELF 바이너리 `stings`를 분석하여 플래그를 추출하는 리버싱 문제입니다.  
프로그램은 사용자 입력을 검증하며, 정답일 경우 "Congrats! The password is the flag." 메시지를 출력합니다.  
- **Target:** ELF 64-bit LSB pie executable (`stings`)  
- **주요 취약점:** 플래그가 바이너리 내부에 하드코딩되어 있으며, 단순한 바이트 단위 비교를 통해 검증되므로 `strings` 또는 디컴파일을 통해 쉽게 추출 가능

---

## 🔍 정찰

### 1) 초기 정찰: 바이너리 정보 확인 및 문자열 추출

문제를 접하자마자 기본적인 정찰을 수행합니다. `file`, `strings`, 실행 테스트를 통해 바이너리의 성격과 동작을 파악합니다.

```bash
ls -la /sandbox/bins/ && file $BINARY && strings $BINARY && echo 'Test input: AAAA' && $BINARY
```

- 바이너리는 **ELF 64-bit**, **not stripped** 상태로, 디버깅 심볼이 포함되어 있어 분석이 용이합니다.
- `strings` 출력에서 다음과 같은 키워드를 발견:
  - `Welcome to the beehive. Enter the password, or you'll get stung!`
  - `Congrats! The password is the flag.`
  - `I'm disappointed. *stings you*`
- 테스트 입력 `AAAA`에 대해 "I'm disappointed" 출력 → 잘못된 입력

또한, `strings` 출력 중 다음과 같은 이상한 문자열들이 눈에 띕니다:
```
jdug|tusH
2oht`5s4H
ou`i2ee4H
o`28c32bH
dH34%(
```
이 값들은 후에 플래그 검증에 사용되는 하드코딩된 값임이 밝혀집니다.

---

### 2) 디컴파일 분석: `main` 함수 로직 파악

바이너리의 핵심 로직을 이해하기 위해 `main` 함수를 디컴파일합니다.

```c
decompile {"function": "main"}
```

디컴파일된 코드를 분석한 결과, 다음과 같은 검증 로직이 존재함을 확인:

- 프로그램은 사용자 입력을 `scanf("%50s")`로 읽어옵니다.
- 입력된 문자열을 바이트 단위로 순회하며, 하드코딩된 배열 값과 비교합니다.
- 비교 조건: `input[i] == (hardcoded_value[i] - 1)`
- 즉, **하드코딩된 값에서 1을 뺀 값이 실제 플래그**

하드코딩된 값들은 스택 변수에 다음과 같이 저장되어 있음:
```c
var_1110h = 0x7375747c6775646a  // little-endian: "jdu|tus" + 'H'?
var_1108h = 0x3473356074686f32  // "2oht`5s4"
var_1100h = 0x346565326960756f  // "ou`i2ee4"
var_10f8h = 0x623233633832606f  // "o`28c32b"
var_10f0h = 0x7e3a37            // ":7~"
```

---

## 🧠 문제 분석

### 검증 로직의 취약성

프로그램은 입력을 단순히 하드코딩된 값 - 1과 비교합니다.  
이는 **아무런 암호화나 난독화 없이 플래그를 바이트 단위로 저장**하고 있음을 의미합니다.

```c
for (int i = 0; i < flag_len; i++) {
    if (input[i] != hardcoded_bytes[i] - 1) {
        puts("I'm disappointed. *stings you*");
        return 1;
    }
}
puts("Congrats! The password is the flag.");
```

따라서, 플래그는 단순히 **하드코딩된 바이트 값에 1을 더한 문자열**입니다.

예:  
- `0x73` → `'s'`, `0x73 - 1 = 0x72` → `'r'` → 입력에는 `'r'`이 필요  
- 하지만 우리는 플래그를 찾고 있으므로, `0x73` → `'s'` → 플래그의 문자는 `'s' + 1 = 't'`? ❌  
- **아니요!** 비교는 `input[i] == (hardcoded[i] - 1)` → 즉, `input[i]`가 `hardcoded[i] - 1`과 같아야 성공  
- 따라서 **정답 = `hardcoded[i] - 1`**

즉, 플래그는 하드코딩된 바이트 배열에서 **각 바이트에 1을 뺀 값**입니다.

---

## 💥 익스플로잇

### 1) 플래그 복원 스크립트 작성

하드코딩된 값을 little-endian으로 바이트 배열로 변환하고, 각 바이트에서 1을 빼면 플래그를 얻을 수 있습니다.

```python
from pwn import *

# 하드코딩된 값들 (little-endian 64비트 정수)
values = [
    0x7375747c6775646a,  # "jdu|tusH" → little-endian 바이트
    0x3473356074686f32,  # "2oht`5s4H"
    0x346565326960756f,  # "ou`i2ee4H"
    0x623233633832606f,  # "o`28c32bH"
    0x7e3a37              # ":7~" (3바이트)
]

# 플래그 바이트 추출
flag_bytes = []
for val in values:
    if val == 0x7e3a37:
        num_bytes = 3
    else:
        num_bytes = 8
    # 리틀엔디언으로 변환 후 각 바이트에서 1을 뺌
    val_bytes = val.to_bytes(num_bytes, 'little')
    for b in val_bytes:
        flag_bytes.append(b - 1)

flag = bytes(flag_bytes)
print("Flag:", flag.decode(errors='replace'))
```

### 2) 실행 결과

```
Flag: ictf{str1ngs_4r3nt_h1dd3n_17b21a69}
```

### 3) 검증: 바이너리에 직접 입력 테스트

pwntools를 사용해 비문자열 바이트도 정확히 전송:

```python
p = process('/sandbox/bins/stings')
p.sendline(flag)
print(p.recvall().decode())
```

출력:
```
Welcome to the beehive. Enter the password, or you'll get stung!
Congrats! The password is the flag.
```

성공!

---

## 🚩 Flag

```
ictf{str1ngs_4r3nt_h1dd3n_17b21a69}