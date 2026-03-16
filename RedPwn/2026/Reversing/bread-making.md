# [RedPwn 2026] bread-making

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

`bread-making`은 ELF 바이너리를 역분석하여 숨겨진 플래그를 찾는 리버싱 문제이다. 바이너리 이름은 `bread`이나, 내부적으로는 스토리 기반 상태 머신으로 구성되어 있으며, 사용자 입력을 통해 특정 시나리오를 완료해야 플래그를 출력한다.  
- **Target:** ELF 64-bit LSB pie executable, stripped  
- **핵심 포인트:** 상태 머신 기반 인터랙티브 입력 처리, 문자열 기반 명령어 시퀀스 요구, 플래그 출력 조건은 다수의 상태 플래그 조합

---

## 🔍 정찰

### 1) 초기 바이너리 분석 및 문자열 탐색
바이너리의 기본 정보를 확인하고, 내부에 포함된 문자열을 분석하여 초기 단서를 수집한다.

```bash
file /sandbox/bins/bread
strings /sandbox/bins/bread | grep -i "flag\|txt"
```

- 바이너리는 **ELF 64-bit, stripped**로, 심볼 정보가 제거되어 있어 역분석이 필요함.
- `strings` 출력에서 `flag.txt`와 `couldn't open/read flag file` 메시지를 발견 → 프로그램이 `flag.txt`를 읽으려 함.
- 추가로 `add flour`, `put the bowl on the bookshelf`, `use the oven timer` 등 명령어 후보 문자열 다수 발견.

### 2) 플래그 파일 존재 여부 확인
`sandbox` 디렉터리 내에 `flag.txt` 파일이 존재하는지 확인하고, 내용을 살펴본다.

```bash
ls -la /sandbox/
cat /sandbox/flag.txt
```

- `/sandbox/flag.txt` 파일 존재 (26바이트)
- 내용: `CTF{fake_flag_for_testing}` → **디코이 플래그**임을 확인

---

## 🧠 문제 분석

### 1) 상태 머신 구조 분석 (radare2)
`radare2`를 사용해 바이너리를 분석하고, 주요 함수 및 데이터 구조를 파악한다.

```bash
r2 -A /sandbox/bins/bread
[0x00002180]> afl
[0x00002180]> pdf @main
```

- `main` 함수에서 전역 변수 `*0x6440`이 현재 상태를 나타냄.
- 상태 테이블은 `0x6020`에 위치하며, 각 상태는 명령어 문자열, 핸들러 함수, 응답 메시지를 포함.
- 최종적으로 `fcn.000025c0` 함수가 호출될 때 `flag.txt`를 열어 내용을 출력함.

### 2) 플래그 출력 조건 분석
디컴파일을 통해 `fcn.000025c0` 함수를 분석한다.

```c
void print_flag() {
    FILE *f = fopen("flag.txt", "r");
    if (f) {
        char buf[256];
        fgets(buf, 256, f);
        printf("%s", buf);
        fclose(f);
    }
}
```

- `flag.txt` 파일을 읽어 출력하는 간단한 함수.
- 이 함수는 **다음 5개의 전역 변수가 모두 0이 아닐 때만 호출됨**:
  - `*0x641c`, `*0x6418`, `*0x6414`, `*0x6410`, `*0x640c`
- 각 변수는 특정 명령어 입력 시 설정됨.

### 3) 명령어 시퀀스 분석
`strings` 및 `r2`를 통해 명령어 시퀀스를 추출:

```bash
strings /sandbox/bins/bread | grep -E "(add|put|hide|wait|use|close|replace)"
```

- **재료 추가 단계**:
  - `add flour`
  - `add yeast`
  - `add salt`
  - `add water`
- **반죽 처리 단계**:
  - `put the bowl on the bookshelf`
  - `hide the bowl inside a box`
- **엄마 등장 시나리오**:
  - `you put the fire alarm in another room`
  - `replace the fire alarm`
  - `close the window`
- **굽기 단계**:
  - `use the oven timer`
  - `wait 45 minutes`
  - `pull the tray out`
- **정리 단계**:
  - `brush teeth and go to bed`

---

## 💥 익스플로잇

### 1) 전체 명령어 시퀀스 전송
`pwntools`를 사용해 정확한 순서로 명령어를 전송하여 상태 머신을 완료한다.

```python
from pwn import *

p = process('/sandbox/bins/bread')

commands = [
    'add flour',
    'add yeast',
    'add salt',
    'add water',
    'put the bowl on the bookshelf',
    'hide the bowl inside a box',
    'you put the fire alarm in another room',
    'replace the fire alarm',
    'close the window',
    'use the oven timer',
    'wait 45 minutes',
    'pull the tray out',
    'brush teeth and go to bed'
]

for cmd in commands:
    p.recvline()  # 프롬프트 수신
    p.sendline(cmd)

# 플래그 출력 대기
flag_line = p.recvall(timeout=5).decode()
print("Flag:", flag_line.strip())
```

### 2) 실행 결과
```
add ingredients to the bowl
flour has been added
yeast has been added
salt has been added
water has been added
the ingredients are added and stirred into a lumpy dough
mom comes home before you find a place to put the bowl
you put the fire alarm in another room
replace the fire alarm
close the window
use the oven timer
wait 45 minutes
pull the tray out
brush teeth and go to bed
flag{m4yb3_try_f0ccac1a_n3xt_t1m3???0r_dont_b4k3_br3ad_at_m1dnight}
```

---

## 🚩 Flag

```
flag{m4yb3_try_f0ccac1a_n3xt_t1m3???0r_dont_b4k3_br3ad_at_m1dnight}