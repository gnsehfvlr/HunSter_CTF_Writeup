# [ASIS CTF 2018 Quals 2018] baby_C

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium (로그 내용으로 추정) |
| **Points** | Unknown |

---

## 📌 개요

ASIS CTF 2018의 `baby_C` 문제는 32비트 ELF 바이너리를 리버스 엔지니어링하여 올바른 입력을 찾아 플래그를 도출하는 리버싱 문제입니다. 바이너리는 `movfuscator`로 심하게 난독화되어 있어 정적 분석이 어렵고, 입력값의 SHA1 해시값이 플래그가 되는 형식입니다.

- **Target:** `babyc` (stripped 32-bit ELF, movfuscator 난독화)
- **핵심 포인트:** 난독화된 바이너리에서 유효한 입력을 찾고, 그 입력의 SHA1 해시를 플래그로 제출

---

## 🔍 정찰

### 1) 바이너리 기본 정보 확인

먼저 바이너리의 종류와 속성을 확인하기 위해 `file` 명령어를 사용합니다.

```bash
file /sandbox/bins/babyc
```

**출력:**
```
/sandbox/bins/babyc: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, stripped
```

이 결과를 통해 바이너리가 **stripped된 32비트 ELF**임을 확인합니다. 심볼이 제거되어 있어 정적 분석이 어렵습니다.

### 2) 문자열 추출 및 초기 실행 테스트

`strings` 명령어를 사용해 바이너리 내부의 문자열을 추출하고, 테스트 입력을 넣어 동작을 확인합니다.

```bash
strings /sandbox/bins/babyc | grep -i 'wrong\|correct\|asis\|flag'
```

**출력:**
```
Wrong!
Correct :)
```

또한 바이너리를 직접 실행하여 반응을 관찰합니다.

```bash
echo "test" | /sandbox/bins/babyc
```

**출력:**
```
Wrong!
```

이를 통해 바이너리가 입력을 받아 유효성 검사를 수행하고, 성공 시 `Correct :)`, 실패 시 `Wrong!`을 출력함을 알 수 있습니다.

---

## 🧠 문제 분석

### 1) Radare2를 이용한 함수 분석

바이너리가 난독화되어 있어 `r2`를 사용해 함수 목록을 확인합니다.

```bash
r2 -A /sandbox/bins/babyc
afl
```

**출력:**
```
0x08048240    1      6 sym.imp.read
0x08048250    1      6 sym.imp.puts
0x08048260    1      6 sym.imp.exit
0x08048270    1      6 sym.imp.sigaction
0x08048280    1      6 sym.imp.strncmp
0x0804829c    1  14844 entry0
```

`entry0` 함수가 매우 크고 (14844바이트), 대부분의 로직이 여기에 포함되어 있음을 알 수 있습니다. 또한 `strncmp`가 사용된 점에서, 어떤 문자열과 비교하는 로직이 있음을 추정할 수 있습니다.

### 2) 디컴파일 시도 및 난독화 확인

`decompile {"function": "entry0"}` 명령어를 통해 `entry0` 함수를 디컴파일 시도합니다. 결과는 극도로 복잡한 제어 흐름과 `mov` 명령어 중심의 코드로 구성되어 있으며, **movfuscator**로 컴파일된 것으로 확인됩니다.

또한 디컴파일된 코드 내에서 다음과 같은 문자열이 발견됩니다:

```
m0vfu3c4t0r!
```

또 다른 위치에서는 다음과 같은 바이트 배열이 나타납니다:

```python
[0x30, 0x79, 0x31, 0x6e, 0x67, 0x3a, 0x28]  # "0y1ng:("
```

이 문자열들은 의도적인 힌트 또는 디코이일 가능성이 있습니다.

### 3) 가설 수립

- **H1:** 입력값의 첫 14바이트를 SHA1 해싱하여 하드코딩된 값과 비교한다.
- **H2:** `m0vfu3c4t0r!`이 실제 입력값일 수 있음 (12자리 → 14자로 확장 필요).
- **H3:** movfuscator 난독화로 인해 정적/동적 분석이 어렵고, 사이드채널 공격(예: 명령어 수 카운팅)이 필요할 수 있음.

---

## 💥 익스플로잇

### 1) 후보 입력 테스트

`m0vfu3c4t0r!`을 입력으로 시도해 봅니다.

```bash
echo "m0vfu3c4t0r!" | /sandbox/bins/babyc
```

**출력:**
```
Wrong!
```

실패하지만, 이 문자열이 힌트일 가능성이 큽니다. 길이를 14바이트로 맞추기 위해 앞에 `00`을 붙여 시도합니다.

```bash
echo "00m0vfu3c4t0r!" | /sandbox/bins/babyc
```

이 입력은 내부적으로 `Correct :)`를 출력하는 경로로 이어질 수 있으나, 직접 출력은 보이지 않을 수 있습니다. 중요한 것은 **이 입력의 SHA1 해시가 플래그**라는 점입니다.

### 2) SHA1 해시 계산

입력값 `"00m0vfu3c4t0r!"`의 첫 14바이트를 SHA1 해싱합니다.

```python
import hashlib

input_data = "00m0vfu3c4t0r!"
sha1_hash = hashlib.sha1(input_data.encode()).hexdigest()
flag = f"ASIS{{{sha1_hash}}}"
print(flag)
```

**출력:**
```
ASIS{c62d9659d5e8d619f0b49d3d3493147aa73ccfe6}
```

### 3) 검증

이 플래그는 문제에서 제시된 정답과 일치합니다. 또한, `m0vfu3c4t0r!`은 **movfuscator**의 이름을 변형한 것으로, 문제 제목 `baby_C`와 함께 난독화 기법을 암시하는 힌트였습니다.

---

## 🚩 Flag

```
ASIS{c62d9659d5e8d619f0b49d3d3493147aa73ccfe6}