# [idek 2026] TUTORIAL - Intro To GDB

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy (로그 내용으로 추정) |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 리눅스 바이너리 역분석을 위한 동적 분석 도구 GDB의 기초를 익히는 튜토리얼 문제입니다. 바이너리는 사용자 입력을 받아 하드코딩된 플래그와 비교하며, 단순한 문자열 비교 방식을 사용합니다.  
- **Target:** `Intro_to_GDB` 바이너리  
- **주요 취약점:** 하드코딩된 플래그 값과의 직접 비교 (strcmp 대신 수동 바이트 비교)

---

## 🔍 정찰

### 1) 파일 구조 확인 및 초기 분석
문제 디렉터리 내 파일을 확인하여 구성 요소를 파악합니다. 바이너리 파일과 `walkthrough.md` 문서가 존재합니다.

```bash
ls -la /sandbox/bins/
```

```
total 32
drwxr-xr-x 2 root root  4096 Mar 10 20:41 .
drwxr-xr-x 1 root root  4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root 16896 Mar 10 20:41 Intro_to_GDB
-rwxr-xr-x 1 root root  2987 Mar 10 20:41 walkthrough.md
```

### 2) Walkthrough 문서 분석
`walkthrough.md`는 GDB와 동적 분석에 대한 기초 설명을 포함하고 있으나, 실제 플래그 관련 정보는 포함하지 않습니다. 문제의 의도는 바이너리 자체를 분석하는 것입니다.

```bash
cat /sandbox/bins/walkthrough.md
```

> 출력 일부:  
> `# Walkthrough of Intro to GDB`  
> `### What is dynamic analysis?`  
> `Dynamic analysis is a type of analysis that requires live execution of the code...`  
> `...basically everything Linux uses. It allows the reverser to step through a program...`  

문서는 GDB 사용법을 설명하지만, 플래그는 포함하지 않으며, ":D"라는 문자열이 바이너리 내에 존재함을 시사합니다.

---

## 🧠 취약점 분석

### 바이너리 동작 확인
테스트 입력을 제공하여 프로그램의 동작을 확인합니다.

```bash
echo "test" | /sandbox/bins/Intro_to_GDB
```

```
Hey, check out my new password checker! :D
Nope, that's not right...
```

프로그램은 플래그를 맞추면 성공 메시지를 출력하며, ":D"는 인사말에 사용됩니다.

### Radare2를 이용한 정적 분석
`r2`를 사용해 함수 목록과 `main` 함수의 디스어셈블리를 분석합니다.

```bash
r2 -A /sandbox/bins/Intro_to_GDB
[0x00001030]> afl
```

```
0x00001165    6    209 main
```

`main` 함수의 디스어셈블리에서 플래그 문자열이 하드코딩된 형태로 저장되어 있음을 확인합니다.

```asm
│           0x0000116d      lea rdi, str.Hey__check_out_my_new_password_checker__:D ; 0x2008
│           0x00001174      call sym.imp.puts
│           0x00001179      lea rax, [var_30h]
│           0x0000117d      mov rsi, rax
│           0x00001180      lea rdi, [0x00002033]       ; "%s"
│           0x00001187      call sym.imp.__isoc99_scanf
│           0x00001191      movabs rax, 0x6d306d7b6b656469  ; 'idek{m0m' (little-endian)
│           0x0000119b      movabs rdx, 0x3368745f7433675f  ; '_g3t_th3' (little-endian)
│           0x000011a5      movabs rax, 0x214172336d34635f  ; '_c4m3rA!' (little-endian)
│           0x000011af      mov word [var_38h], 0x7d       ; '}'
│           0x000011b5      mov byte [var_36h], 0          ; null
```

이 값들을 리틀 엔디언으로 바이트로 변환하면 플래그 조각을 재구성할 수 있습니다.

---

## 💥 익스플로잇

### Python을 이용한 플래그 재구성
하드코딩된 64비트 정수 값을 리틀 엔디언으로 바이트로 변환하여 플래그를 조합합니다.

```python
import struct

# 하드코딩된 값들
part1 = struct.pack('<Q', 0x6d306d7b6b656469)  # 'idek{m0m'
part2 = struct.pack('<Q', 0x3368745f7433675f)  # '_g3t_th3'
part3 = struct.pack('<Q', 0x214172336d34635f)  # '_c4m3rA!'
part4 = b'}'

flag = part1 + part2 + part3 + part4
print(f"Flag: {flag.decode()}")
```

**출력:**
```
Flag: idek{m0m_g3t_th3_c4m3rA!}
```

### 플래그 검증
재구성된 플래그를 바이너리에 입력하여 성공 메시지를 확인합니다.

```bash
echo "idek{m0m_g3t_th3_c4m3rA!}" | /sandbox/bins/Intro_to_GDB
```

```
Hey, check out my new password checker! :D
GGs, you got it!
```

성공 메시지가 출력되며, 플래그가 유효함이 확인됩니다.

---

## 🚩 Flag

```
idek{m0m_g3t_th3_c4m3rA!}