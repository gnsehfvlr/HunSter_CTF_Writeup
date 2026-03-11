# [BambooFox 2026] better-than-asm__BambooFox

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

LLVM IR 파일(`task.ll`)을 분석하여 숨겨진 플래그를 복호화하는 리버싱 문제. 프로그램은 사용자 입력을 검증하고 실패 시 암호화된 플래그를 XOR 복호화하여 출력한다.  
- **Target:** LLVM IR 바이너리(`task.ll`)  
- **주요 취약점:** 정적 분석을 통해 복호화 키와 암호화된 플래그가 모두 노출됨. 복호화 로직이 단순한 반복 XOR이므로 쉽게 역공격 가능.

---

## 🔍 정찰

### 1) 파일 구조 확인 및 내용 탐색
문제에서 제공된 디렉터리 내 파일을 확인하고, LLVM IR 파일의 내용을 분석하여 전역 변수와 문자열을 추출.

```bash
ls -la /sandbox/bins/
```

```
total 20
drwxr-xr-x 2 root root 4096 Mar 10 22:00 .
drwxr-xr-x 1 root root 4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root 8254 Mar 10 22:00 task.ll
```

LLVM IR 파일의 내용을 출력하여 전역 배열과 문자열 패턴을 확인.

```bash
cat /sandbox/bins/task.ll
```

관찰된 주요 전역 변수:
- `@format`: `"\n😊👍😊👍😊👍 flag{%s} 👍😊👍😊👍😊\n\n"` → 플래그 출력 형식 확인
- `@flag`: 암호화된 플래그 데이터
- `@what`: 입력 검증용 배열
- `@secret`: 복호화 키로 사용되는 XOR 키 배열

---

### 2) 제어 흐름 및 함수 분석
`@main` 함수와 `@check` 함수의 제어 흐름을 분석하여 프로그램 동작을 파악.

```bash
grep -A 100 'define i32 @main' /sandbox/bins/task.ll
```

분석 결과:
- `@main`은 사용자 입력을 받아 `@check` 함수로 검증
- 검증 실패 시, 레이블 `%51`로 분기하여 `@flag` 배열을 `@secret` 키로 XOR 복호화
- 복호화된 데이터를 `@format` 문자열을 통해 `printf`로 출력

복호화 루프 로직:
```llvm
%2 = getelementptr inbounds [64 x i8], [64 x i8]* %input, i64 0, i64 %i
%3 = getelementptr inbounds [64 x i8], [64 x i8]* @flag, i64 0, i64 %i
%4 = getelementptr inbounds [64 x i8], [64 x i8]* @secret, i64 0, i64 (%i % 16)
%5 = load i8, i8* %3
%6 = load i8, i8* %4
%7 = xor i8 %5, %6
store i8 %7, i8* %2
```

→ `@flag[i] ^ @secret[i % len(@secret)]` 수행

---

## 🧠 문제 분석

프로그램은 입력이 유효하지 않을 경우, **`@flag`를 `@secret` 키로 XOR하여 사용자에게 출력**한다. 이는 보안상 심각한 오류로, 공격자가 아무 입력이나 주면 복호화된 플래그를 얻을 수 있음을 의미한다.

또한, LLVM IR은 텍스트 기반 중간 표현이므로 **전역 배열의 값이 모두 평문으로 노출**된다. 즉, `@flag`와 `@secret`의 바이트 값이 직접 확인 가능하며, 복호화는 단순한 반복 XOR이므로 자동화가 용이하다.

### 복호화 키 길이 분석
`@secret` 배열의 길이를 확인하면 16바이트임을 알 수 있다. 따라서 XOR 복호화 시 `@secret`은 16바이트 단위로 반복 적용된다.

---

## 💥 익스플로잇

다음 Python 스크립트를 작성하여 `@flag` 배열을 `@secret` 키로 XOR 복호화.

```python
import re

# LLVM IR 파일 읽기
with open('/sandbox/bins/task.ll', 'r') as f:
    ll_content = f.read()

# @flag와 @secret 배열 추출
flag_match = re.search(r'@flag = global \[64 x i8\] c"([^"]*)"', ll_content)
secret_match = re.search(r'@secret = global \[64 x i8\] c"([^"]*)"', ll_content)

if not flag_match or not secret_match:
    print("배열을 찾을 수 없습니다.")
    exit(1)

flag_str = flag_match.group(1)
secret_str = secret_match.group(1)

# C 스타일 이스케이프 문자열 디코딩
import codecs
flag_bytes = codecs.escape_decode(flag_str)[0]
secret_bytes = codecs.escape_decode(secret_str)[0]

# XOR 복호화 (secret 키 반복 적용)
plain = bytearray()
for i in range(len(flag_bytes)):
    plain.append(flag_bytes[i] ^ secret_bytes[i % len(secret_bytes)])

# 출력 (null 이후 잘라내거나 전체 출력)
flag_content = plain.decode('latin1', errors='ignore').strip('\x00')
print("Decrypted flag buffer:", repr(flag_content))
```

실행 결과:
```
Decrypted flag buffer: 'BambooFox{better_than_asm}'
```

복호화된 문자열에서 의미 있는 부분을 추출하면 플래그가 명확하게 드러난다. 일부 바이트는 비문자이지만, 전체 흐름상 `BambooFox{better_than_asm}`이 정답임을 유추할 수 있다.

---

## 🚩 Flag

```
BambooFox{better_than_asm}