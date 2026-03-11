# [ShadowCTF 2026] Warm-up-rev

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 단순하지만 심리적 장벽을 유도하는 리버싱 문제로, `sleep(3600)`을 통해 플래그 출력을 1시간 뒤로 지연시키는 트릭을 사용합니다. 실제 플래그는 문자열 난독화나 복잡한 암호화 없이 고정된 문자열로 존재합니다.
- **Target:** 64비트 ELF 바이너리 (`Intro`)
- **주요 취약점:** 지나치게 긴 `sleep()` 호출을 통한 사용자 인내심 유도 (시간 기반 지연)

---

## 🔍 정찰

### 1) 바이너리 정보 확인 및 기본 분석

먼저 샌드박스 내 파일 목록을 확인하고, 주어진 바이너리의 속성을 분석합니다. `file` 명령어를 통해 ELF 형식임을 확인합니다.

```bash
ls -la /sandbox/bins/
```
```
total 28
drwxr-xr-x 2 root root  4096 Mar 10 21:17 .
drwxr-xr-x 1 root root  4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root 16704 Mar 10 21:17 Intro
```

```bash
file /sandbox/bins/Intro
```
```
/sandbox/bins/Intro: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=62c32f555e0a8c2e29c1415e08813c1f7924c805, for GNU/Linux 3.2.0, not stripped
```

### 2) 문자열 추출을 통한 초기 단서 확보

바이너리에서 출력 가능한 문자열을 추출하여 플래그 관련 단서를 찾습니다.

```bash
strings /sandbox/bins/Intro
```
```
you need patience to get the flag.
{steppingstone}
...
```

두 가지 핵심 문자열이 발견됩니다:
- `you need patience to get the flag.` — 사용자에게 "기다리라"는 메시지
- `{steppingstone}` — 플래그 후보이나 형식이 불완전해 보임

---

## 🧠 문제 분석

### 1) 정적 분석을 통한 제어 흐름 확인

`radare2`를 사용해 `main` 함수를 디스어셈블하여 실행 흐름을 분석합니다.

```r2
pdf @main
```
```
│           0x00001159      lea rdi, str.you_need_patience_to_get_the_flag.
│           0x00001160      call sym.imp.puts
│           0x00001165      mov edi, 0xe10        ; 3600 seconds (1 hour)
│           0x0000116a      mov eax, 0
│           0x0000116f      call sym.imp.sleep
│           0x00001174      lea rdi, str.steppingstone
│           0x0000117b      mov eax, 0
│           0x00001180      call sym.imp.printf
```

분석 결과, 다음과 같은 흐름이 확인됩니다:
1. `"you need patience to get the flag."` 출력
2. `sleep(3600)` 호출 (1시간 대기)
3. `"{steppingstone}"` 출력

이로 인해 플래그는 단순히 **시간이 지나면 출력되는 문자열**임을 알 수 있습니다. `not stripped` 상태이므로 심볼이 남아 있어 분석이 용이합니다.

---

## 💥 익스플로잇

### 1) `LD_PRELOAD`를 이용한 `sleep()` 우회

1시간을 기다리는 것은 비현실적이므로, `sleep()` 함수를 무시하도록 동적 라이브러리를 인젝션합니다. `pwntools`와 함께 `LD_PRELOAD`를 사용해 `sleep()`을 즉시 반환하게 조작합니다.

```python
from pwn import *
import os

code = '''
#include <stdio.h>
unsigned int sleep(unsigned int seconds) {
    return 0;
}
'''

with open('/tmp/fake_sleep.c', 'w') as f:
    f.write(code)

os.system('gcc -shared -fPIC -o /tmp/fake_sleep.so /tmp/fake_sleep.c')

context.binary = '/sandbox/bins/Intro'
env = {'LD_PRELOAD': '/tmp/fake_sleep.so'}
p = process('/sandbox/bins/Intro', env=env)
print(p.recvall().decode())
p.close()
```

### 실행 결과

```
you need patience to get the flag.
{steppingstone}
```

출력된 문자열 `{steppingstone}`는 플래그 형식이 아니지만, 문제의 맥락상 실제 플래그는 `CTF{steppingstone}`임을 추측할 수 있습니다. 이후 제출을 통해 정답임이 확인됩니다.

---

## 🚩 Flag

```
CTF{steppingstone}
```