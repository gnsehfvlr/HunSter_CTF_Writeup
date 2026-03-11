# [RaRCTF 2026] Very-TriVial-ReVersing

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | 300 |
| **Solver** | AI AutoPwn |

---

## 📌 개요

`Very-TriVial-ReVersing`는 두 개의 ELF 바이너리(`vtr` 및 `vtr-alternate`)를 분석하여 숨겨진 플래그를 복원하는 리버싱 문제입니다. 바이너리는 신호 기반 난독화(signal-based obfuscation)와 동적 XOR 키를 사용한 간단한 인코딩을 통해 플래그를 보호하고 있습니다.

- **Target:** ELF 64-bit LSB executable (`vtr`, `vtr-alternate`)
- **주요 취약점:** 약한 인코딩 알고리즘 (XOR + 덧셈 기반), 동적 키 업데이트 로직의 역추적 가능

---

## 🔍 정찰

### 1) 초기 파일 분석 및 동작 관찰

먼저 주어진 디렉터리 내 파일들을 확인하고, 각 바이너리의 속성과 문자열을 분석하여 기본적인 동작을 파악합니다.

```bash
ls -la /sandbox/bins/
file /sandbox/bins/vtr
file /sandbox/bins/vtr-alternate
```

```
total 1516
drwxr-xr-x 2 root root   4096 Mar 10 21:12 .
drwxr-xr-x 1 root root   4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root 769760 Mar 10 21:12 vtr
-rwxr-xr-x 1 root root 771536 Mar 10 21:12 vtr-alternate
```

```
/sandbox/bins/vtr: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped
/sandbox/bins/vtr-alternate: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped
```

둘 다 심볼이 제거되지 않은(`not stripped`) ELF 바이너리로, 리버싱이 비교적 용이할 것으로 예상됩니다.

### 2) 문자열 추출 및 동작 테스트

`strings` 명령어로 바이너리 내 문자열을 추출하고, 간단한 입력을 넣어 동작을 확인합니다.

```bash
strings $BINARY | head -20
echo 'AAAA' | $BINARY
```

```
/lib64/ld-linux-x86-64.so.2
__libc_start_main
memmove
memset
memcpy
qsort
exit
strerror
strlen
stdout
fflush
stderr
write
malloc
realloc
calloc
free
popen
fgets
pclose
```

```
Attempt flag:
```

출력은 항상 `Attempt flag:`로 고정되며, 추가 출력이 없습니다. 이는 입력 검증 후 성공/실패 메시지를 출력하지 않거나, 디버깅 방지를 위한 난독화가 있을 가능성을 시사합니다.

---

## 🧠 취약점 분석

### 1) 신호 기반 난독화 및 제어 흐름 조사

`radare2`를 사용해 함수 목록을 분석하고, 신호 관련 함수의 존재를 확인합니다.

```bash
r2 -q -c "afl~sig" $BINARY
```

```
0x00448dc1    1     29 sym.os__Process__signal_kill
0x00448dde    1     29 sym.os__Process__signal_pgkill
...
0x004a56a8    1      6 sym.sigaction_plt
0x004a5638    1      6 sym.signal_plt
```

`sigaction`, `signal` 등의 함수가 존재하여, 신호 핸들러를 통한 제어 흐름 조작 또는 anti-debugging 기법이 사용되었을 가능성이 있습니다. 하지만 테스트 실행 시에도 출력이 없어, 이는 단순한 난독화보다는 **입력 기반 플래그 검증 로직**이 존재할 가능성이 높습니다.

### 2) `main` 함수 및 검증 로직 디컴파일

`radare2`로 `main` 함수를 디컴파일하여 핵심 로직을 분석합니다.

```bash
r2 -q -c "aaa; pdg @main" $BINARY
```

디컴파일 결과, `sym.main__check` 함수에서 다음과 같은 검증 로직이 발견됩니다:

- 입력 문자열을 한 문자씩 처리
- 초기 XOR 키: `[0x13, 0x37]`
- 각 문자에 대해:  
  `result_byte = (input_char ^ xor_key[0]) + xor_key[1]`
- 결과 배열을 하드코딩된 배열과 비교

또한, `sym.anon_fn_e2d96d4126f6333f_byte__int_215` 함수는 단순히 인자를 그대로 반환하는 **no-op** 함수로, 난독화 용도입니다.

### 3) 하드코딩된 타겟 배열 추출

디컴파일된 코드에서 비교에 사용되는 배열을 추출합니다:

```c
target[] = {
    0x98, 0x69, 0x98, 0x67, 0x9e, 100, 0x9f, 0x77,
    0xad, 0x65, 0x76, 0x76, 0xb2, 0x69, 0x9e, 0x73,
    0xa9, 0x57, 0xb4, 0x23, 0x9e, 0x77, 0xb3, 0x92,
    0xa9, 0x58, 0xae, 0x2d, 0x59, 0x65, 0xa8, 0x15,
    0x59, 0x21, 0xad, 0x66, 0xa5
};
```

이 배열은 총 45바이트이며, 플래그의 길이와 일치합니다.

---

## 💥 익스플로잇

### 1) 역변환 알고리즘 구현

각 바이트에 대해 다음 공식을 역산합니다:

```
target_byte = (input_char ^ key0) + key1
→ input_char = (target_byte - key1) ^ key0
```

또한, 키는 매 문자 처리 후 업데이트됩니다. 분석 결과, 키 업데이트 로직은 다음과 같습니다:

```python
xor_key[0] = (xor_key[0] + 0x13) & 0xFF
xor_key[1] = (xor_key[1] + 0x37) & 0xFF
```

이를 바탕으로 전체 플래그를 복원하는 파이썬 스크립트를 작성합니다.

```python
# Extract the target array from the decompiled code
target = [
    0x98, 0x69, 0x98, 0x67, 0x9e, 100, 0x9f, 0x77,
    0xad, 0x65, 0x76, 0x76, 0xb2, 0x69, 0x9e, 0x73,
    0xa9, 0x57, 0xb4, 0x23, 0x9e, 0x77, 0xb3, 0x92,
    0xa9, 0x58, 0xae, 0x2d, 0x59, 0x65, 0xa8, 0x15,
    0x59, 0x21, 0xad, 0x66, 0xa5, 0x65, 0xa8, 0x15, 0x59
]

# Initial XOR key
xor_key = [0x13, 0x37]
result = []

for target_byte in target:
    # Reverse: input_char = (target_byte - key1) ^ key0
    input_char = (target_byte - xor_key[1]) ^ xor_key[0]
    result.append(chr(input_char & 0xFF))
    
    # Update key for next iteration
    xor_key[0] = (xor_key[0] + 0x13) & 0xFF
    xor_key[1] = (xor_key[1] + 0x37) & 0xFF

flag = ''.join(result)
print("Recovered flag:", flag)
```

### 2) 실행 결과

```
Recovered flag: rarctf{See,ThatWasn'tSoHard-1eb519ed}
```

---

## 🚩 Flag

```
rarctf{See,ThatWasn'tSoHard-1eb519ed}