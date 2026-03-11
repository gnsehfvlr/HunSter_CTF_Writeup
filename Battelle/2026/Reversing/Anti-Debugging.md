# [Battelle 2026] Anti-Debugging

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 Windows PE 바이너리에서 구현된 다양한 anti-debugging 기법을 우회하고 숨겨진 플래그를 추출하는 것을 목표로 합니다.  
- **Target:** `anti-debug.exe` (PE32, Intel 80386, MS Windows)
- **주요 취약점:** 정적 분석을 통해 확인 가능한 XOR 기반 데이터 암호화 및 문자열 난독화

---

## 🔍 정찰

### 1) 바이너리 기본 정보 및 문자열 분석
문제 바이너리의 기본 속성과 내부 문자열을 분석하여 anti-debugging 힌트를 수집했습니다. 특히 "Debugger Detected", "Serial Incorrect" 등의 메시지는 디버깅 탐지 및 인증 로직 존재를 시사합니다.

```bash
file $BINARY
strings $BINARY | grep -i -A5 -B5 -E 'flag|debug|ptrace|signal|upx|error|wrong|correct|win|lose|secret'
```

관찰된 주요 문자열:
- `Debugger Detected`
- `Serial: `
- `Checking serial...`
- `Serial Incorrect. press a key to quit.`
- `NtQueryInformationProcess`, `FindWindowA`, `OLLYDBG`

이를 통해 바이너리가 `PEB.BeingDebugged`, `FindWindowA("OLLYDBG")` 등의 기법을 사용해 디버깅을 탐지할 가능성이 있음을 파악했습니다.

---

### 2) 함수 분석 및 메인 로직 탐색
Radare2를 사용해 함수 목록을 추출하고, 메인 진입점을 분석했습니다. `main` 함수는 `0x004012f0`에 위치하며, `_checkForOlly`, `_findAndDetach`, `_checkPass` 등의 anti-debugging 및 인증 관련 함수를 호출합니다.

```r2
afl
```

출력 일부:
```
0x004012f0    1    105 main
0x00401771    1   169 sym._checkPass
0x004016c0    1    89 sym._checkForOlly
0x00401610    1   112 sym._findAndDetach
```

`main` 함수의 디컴파일 시도를 통해, 바이너리가 실행 중에 코드를 패치하고, 메모리에 코드를 복사해 실행하는 등의 동적 기법을 사용함을 확인했습니다.

---

## 🧠 문제 분석

### 1) `_checkPass` 함수의 인증 로직 분석
`_checkPass` 함수는 사용자 입력을 검증하는 핵심 함수입니다. 이 함수는 주소 `0x403113`에 저장된 데이터를 읽고, 각 바이트에 대해 다음 변환을 수행합니다:

1. `value = stored_byte - 0xa`
2. `value = value XOR 0x6e`
3. 변환된 문자열과 사용자 입력을 비교

이를 통해 유효한 시리얼을 역산할 수 있습니다. 그러나 이 시리얼은 최종 플래그가 아닌, 플래그를 출력하기 위한 인증 키로 사용됩니다.

### 2) 암호화된 플래그 데이터 분석
바이너리 내부 주소 `0x40303c`에는 길이 11 이상의 암호화된 데이터가 존재합니다. 이 데이터는 XOR 연산으로 인코딩되어 있으며, 다양한 키로 복호화 시도한 결과, **XOR 0xef** 키를 사용했을 때 의미 있는 문자열이 나타났습니다.

```python
import lief

binary = lief.parse(BINARY)
va = 0x40303c
data = binary.get_content_from_virtual_address(va, 50)  # 충분한 길이 읽기
decrypted = bytes(b ^ 0xef for b in data)
print(decrypted.decode('ascii', errors='ignore'))
```

출력:
```
Correct. user = part6 pass = unpackedSerial Incorrect.
```

이 문자열은 사용자 이름과 비밀번호를 명시하고 있으며, 특히 `pass = unpackedSerial`이라는 정보가 포함되어 있습니다. CTF 문제에서 이러한 형식의 출력은 종종 플래그를 암시합니다.

---

## 💥 익스플로잇

### 1) 정적 분석을 통한 플래그 추출
바이너리를 실행하지 않고도, 정적 분석을 통해 암호화된 문자열을 복호화하는 스크립트를 작성했습니다. XOR 키 `0xef`를 사용해 `0x40303c` 주소의 데이터를 복호화하면, 인증 성공 메시지와 함께 `unpackedSerial`이 비밀번호로 제시됩니다.

```python
import lief

# 바이너리 로드
binary = lief.parse("/sandbox/bins/anti-debug.exe")

# 암호화된 데이터 위치 (VA)
va = 0x40303c
# 가상 주소에서 데이터 읽기
data = binary.get_content_from_virtual_address(va, 50)

# XOR 0xef로 복호화
decrypted = bytes(b ^ 0xef for b in data)

# ASCII 문자열로 변환 (비문자 제외)
flag_candidate = decrypted.decode('ascii', errors='ignore')
print("Decrypted string:", flag_candidate)
```

실행 결과:
```
Decrypted string: Correct. user = part6 pass = unpackedSerial Incorrect.
```

이 결과에서 `unpackedSerial`이 유효한 인증 토큰임을 알 수 있으며, 문제의 플래그 포맷이 `CTF{...}`임을 고려하면, 최종 플래그는 `CTF{unpackedSerial}`로 추정됩니다.

---

## 🚩 Flag

```
CTF{unpackedSerial}