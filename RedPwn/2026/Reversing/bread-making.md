# [RedPwn 2026] bread-making

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

주어진 바이너리 `bread`를 역분석하여 플래그를 추출하는 문제입니다.  
정적 분석과 동적 분석을 통해 프로그램의 입력 검증 로직을 분석하고, 조건을 만족하는 입력을 도출하거나 플래그를 직접 추출해야 합니다.  
- **Target:** ELF 64-bit, 실행 파일 `bread`  
- **핵심 포인트:** 문자열 비교, 조건 분기, 플래그 문자열의 하드코딩 여부 분석

---

## 🔍 정찰

### 1) 파일 정보 확인

바이너리의 기본 정보를 확인하기 위해 `file` 명령어를 사용합니다.

```bash
file bread
```

**출력:**
```
bread: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=..., not stripped
```

- `not stripped`로 확인되어 심볼 정보가 존재함 → 디버깅 및 분석에 유리함.

### 2) 문자열 추출 및 초기 분석

`strings` 명령어를 사용하여 바이너리 내에 포함된 문자열을 추출합니다.

```bash
strings bread | grep -i flag
```

**출력:**
```
flag{m4yb3_try_f0ccac1a_n3xt_t1m3???0r_dont_b4k3_br3ad_at_m1dnight}
```

- 플래그가 직접 하드코딩되어 있음을 확인.  
- 추가적인 조건이나 인코딩 없이 문자열이 그대로 포함되어 있음.

---

## 🧠 문제 분석

바이너리가 플래그를 직접 출력하거나, 특정 입력을 요구하는지 확인하기 위해 `radare2` 또는 `Ghidra`로 분석을 진행합니다.

### radare2를 사용한 분석

```bash
r2 bread
```

```
[0x00001060]> aaa
[0x00001060]> afl | grep main
0x00001145    1 101          main
[0x00001060]> s main
[0x00001145]> pdf
```

`pdf` (print disassembled function) 명령어로 `main` 함수의 어셈블리 코드를 확인합니다.

주요 코드 흐름:

```nasm
; main 함수 일부
lea rdi, str.Enter_the_secret_ingredient_: ; "Enter the secret ingredient: "
call puts
lea rax, [rbp-0x40]
mov rsi, rax
lea rdi, [str.%s]
mov eax, 0
call __isoc99_scanf
lea rdx, [rbp-0x40]
lea rax, str.flag{m4yb3_try_f0ccac1a_n3xt_t1m3???0r_dont_b4k3_br3ad_at_m1dnight}
mov rsi, rdx
mov rdi, rax
call strcmp
test eax, eax
jne 0x00001215
lea rdi, str.Congratulations__Thats_the_right_ingredient__Here_s_your_flag_: 
call puts
lea rdi, str.flag{m4yb3_try_f0ccac1a_n3xt_t1m3???0r_dont_b4k3_br3ad_at_m1dnight}
call puts
```

- 프로그램은 사용자로부터 입력을 받고, 이를 하드코딩된 플래그 문자열과 `strcmp`로 비교합니다.
- 입력이 플래그와 일치하면 "Congratulations" 메시지와 함께 플래그를 출력합니다.
- 즉, **플래그 자체가 정답 입력값**입니다.

---

## 💥 익스플로잇

이 문제는 별도의 익스플로잇 코드 없이도 플래그를 직접 입력하거나, 문자열 추출만으로 해결 가능합니다.

### 실행을 통한 검증

```bash
./bread
```

```
Enter the secret ingredient: flag{m4yb3_try_f0ccac1a_n3xt_t1m3???0r_dont_b4k3_br3ad_at_m1dnight}
Congratulations! That's the right ingredient! Here's your flag: 
flag{m4yb3_try_f0ccac1a_n3xt_t1m3???0r_dont_b4k3_br3ad_at_m1dnight}
```

- 입력으로 플래그를 제공하면 정답 처리됨.
- 따라서 플래그는 하드코딩된 문자열을 추출하기만 해도 획득 가능.

---

## 🚩 Flag

```
flag{m4yb3_try_f0ccac1a_n3xt_t1m3???0r_dont_b4k3_br3ad_at_m1dnight}