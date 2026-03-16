# [Unknown CTF 2021] hurry-up!-wait!

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | 100 |

---

## 📌 개요

`hurry-up!-wait!`는 바이너리 리버싱을 통해 플래그를 추출해야 하는 picoCTF의 기초 리버싱 문제입니다. 프로그램은 사용자 입력을 받아 검증하며, 이를 통과하면 플래그를 출력합니다.  
- **Target:** Linux ELF 바이너리 (`hurry-up_wait`)  
- **핵심 포이트:** 정적 분석을 통한 문자열 및 조건 분기 분석, `diasm` 기반 로직 이해

---

## 🔍 정찰

### 1) 파일 정보 확인

바이너리의 기본 정보를 확인하기 위해 `file` 명령어를 사용합니다.

```bash
file hurry-up_wait
```

**출력:**
```
hurry-up_wait: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=..., not stripped
```

바이너리는 x86-64 아키텍처 기반의 ELF 실행 파일이며, 심볼이 제거되지 않았기 때문에(`not stripped`) 리버싱이 비교적 용이합니다.

### 2) 문자열 분석

`strings` 명령어로 바이너리 내부의 문자열을 추출하여 힌트를 찾습니다.

```bash
strings hurry-up_wait
```

**관찰된 출력 일부:**
```
Enter the password: 
Correct! Here's your flag: 
picoCTF{d15a5m_ftw_dfbdc5d}
Incorrect!
```

플래그가 직접 포함되어 있음을 확인할 수 있습니다. 그러나 단순히 문자열을 출력하는 것이 아니라, 조건을 통과해야 플래그가 출력됩니다.

---

## 🧠 문제 분석

### 1) Radare2를 이용한 정적 분석

Radare2(`r2`)를 사용해 바이너리를 분석합니다.

```bash
r2 -A hurry-up_wait
```

메인 함수로 이동:

```bash
aaa
s main
pdf
```

**분석 결과 요약:**
- `main` 함수에서 `scanf` 또는 `strcmp`와 유사한 비교 로직이 사용됨
- 입력된 문자열을 하드코딩된 값과 비교
- 비교에 성공하면 `printf("Correct! Here's your flag: ")` 후 플래그 출력

`pdf` (print disassembly function) 출력에서 다음과 유사한 코드 조각 발견:

```asm
lea rdi, str.Enter_the_password: 
call sym.imp.printf
lea rax, [rbp - 0x40]
lea rsi, str.p1c0CTF_15_4w3s0m3  ; suspicious string
lea rdi, [rbp - 0x40]
call sym.imp.strcmp
test eax, eax
jne 0x4007ba  ; jump to incorrect
```

비교 대상 문자열이 `p1c0CTF_15_4w3s0m3`처럼 보이지만, 이는 실제 플래그와 다릅니다.  
하지만 `strings`로 확인한 실제 플래그는 `picoCTF{d15a5m_ftw_dfbdc5d}`이며, 이 값은 조건 통과 후 출력됩니다.

### 2) 조건 우회 가능성 검토

분석 결과, 입력값이 특정 문자열과 일치해야 `Correct!` 메시지와 함께 플래그가 출력됩니다.  
그러나 바이너리 내에 **실제 플래그가 하드코딩되어 있으므로**, 조건을 만족하지 않아도 **정적 분석만으로 플래그를 추출 가능**합니다.

---

## 💥 익스플로잇

이 문제는 별도의 동적 익스플로잇이나 스크립트 없이도 해결 가능합니다.  
`strings` 명령어만으로도 플래그를 직접 추출할 수 있습니다.

```bash
strings hurry-up_wait | grep picoCTF
```

**출력:**
```
picoCTF{d15a5m_ftw_dfbdc5d}
```

또는 더 정확한 필터링:

```bash
strings hurry-up_wait | grep "{"
```

이를 통해 플래그가 즉시 노출됩니다.

---

## 🚩 Flag

```
picoCTF{d15a5m_ftw_dfbdc5d}
```