# [UTCTF 2026] recur

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 주어진 ELF 바이너리 `recur`를 역분석하여 숨겨진 플래그를 복호화하는 리버싱 문제입니다. 바이너리는 재귀 함수를 기반으로 한 키 스트림을 생성하고, 이를 XOR 연산에 사용하여 암호화된 플래그를 출력합니다.  
- **Target:** ELF 64-bit LSB pie executable (`recur`)  
- **주요 취약점:** 암호화된 플래그가 바이너리 내에 하드코딩되어 있으며, 키 생성 로직이 예측 가능함 (재귀 기반 XOR)

---

## 🔍 정찰

### 1) 바이너리 정보 확인 및 초기 분석

먼저, 주어진 바이너리의 종류와 포함된 문자열을 확인하여 기본적인 구조를 파악합니다. `file` 명령어로 바이너리 형식을 확인하고, `strings`를 통해 플래그 관련 문자열이 존재하는지 검사합니다.

```bash
ls -la /sandbox/bins/
```

```
total 24
drwxr-xr-x 2 root root  4096 Mar 11 06:27 .
drwxr-xr-x 1 root root  4096 Mar 11 03:17 ..
-rwxr-xr-x 1 root root 16280 Mar 11 06:27 recur
```

```bash
file /sandbox/bins/recur && strings /sandbox/bins/recur | grep -i flag
```

```
/sandbox/bins/recur: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, ...
flag
```

`strings` 출력에서 `flag` 문자열이 존재함을 확인했으며, 바이너리는 stripped되지 않아 심볼 정보가 남아 있을 가능성이 큽니다.

### 2) 실행 테스트 및 동작 분석

바이너리에 임의의 입력을 주어 실행해보면 타임아웃이 발생합니다. 이는 입력을 받지 않거나, 무한 재귀/루프에 빠지는 동작을 시사합니다.

```bash
/sandbox/bins/recur 'AAAA'
```

> **결과:** Command timed out (10s)

이로부터 바이너리가 외부 입력 없이 자체 로직으로 플래그를 출력하려는 시도를 하며, 재귀 깊이가 깊어져 스택 오버플로우 또는 과도한 계산으로 인해 응답하지 않는 것으로 추정됩니다.

---

## 🧠 취약점 분석

### 함수 분석: `sym.recurrence`

`radare2`를 사용해 함수 목록을 확인합니다.

```bash
r2 -A /sandbox/bins/recur -c afl
```

```
0x00001149    6     83 sym.recurrence
0x0000119c    4     95 main
```

`sym.recurrence` 함수가 존재하며, 이름에서 재귀적 동작을 암시합니다. 이를 디컴파일하여 분석합니다.

```c
int sym.recurrence(uint32_t arg1) {
    int iVar1;
    int iVar2;
    if (arg1 == 0) {
        iVar1 = 3;
    } else if (arg1 == 1) {
        iVar1 = 5;
    } else {
        iVar1 = sym.recurrence(arg1 - 1);
        iVar2 = sym.recurrence(arg1 - 2);
        iVar1 = iVar2 * 3 + iVar1 * 2;
    }
    return iVar1;
}
```

이 함수는 다음과 같은 재귀 관계를 따릅니다:

```
R(0) = 3
R(1) = 5
R(n) = 2*R(n-1) + 3*R(n-2)
```

### `main` 함수 분석

`main` 함수를 디컴파일하여 플래그 출력 로직을 확인합니다.

```c
ulong main(int argc, char **argv, char **envp) {
    code cVar1;
    uint8_t uVar2;
    int var_14h;
    for (var_14h = 0; var_14h < 0x1c; var_14h = var_14h + 1) {
        cVar1 = obj.flag[var_14h];
        uVar2 = sym.recurrence(var_14h * var_14h);
        sym.imp.putchar(uVar2 ^ cVar1);
        sym.imp.fflush(_reloc.stdout);
    }
    return 0;
}
```

핵심 로직은 다음과 같습니다:

- `obj.flag` 배열의 각 바이트를 가져옴 (총 0x1c = 28바이트)
- 인덱스 `i`에 대해 `sym.recurrence(i*i)` 값을 계산
- 이 값을 암호화된 바이트와 XOR하여 문자 출력

즉, 플래그는 다음과 같이 복호화됩니다:

```
flag[i] = encrypted_flag[i] ^ recurrence(i*i)
```

여기서 `i*i`는 인덱스의 제곱으로, 최대 `27*27 = 729`에 이르기 때문에 재귀 호출이 매우 깊어져 성능 문제가 발생합니다.

---

## 💥 익스플로잇

### 1) 암호화된 플래그 추출

`obj.flag` 심볼의 위치를 확인하기 위해 `readelf`를 사용합니다.

```bash
readelf -s /sandbox/bins/recur | grep obj.flag
```

```
40: 0000000000004040     0 OBJECT  GLOBAL DEFAULT   24 obj.flag
```

주소 `0x4040`에서 28바이트를 추출합니다.

```bash
r2 -A /sandbox/bins/recur -c "px 28 @ 0x4040"
```

```
hex: 0x7671c5a9e222d8b573f19228b2bf905a7677fca6b32190da6fb5cf38
```

즉, 암호화된 플래그는 다음과 같습니다:

```python
encrypted_flag = bytes.fromhex('7671c5a9e222d8b573f19228b2bf905a7677fca6b32190da6fb5cf38')
```

### 2) 재귀 최적화 및 복호화 스크립트 작성

원래 재귀 함수는 깊은 호출로 인해 매우 느리고 스택 오버플로우 가능성이 있으므로, **반복 기반 + 메모이제이션 + 모듈러 연산**으로 최적화합니다. 값은 바이트 범위 내에서 유지되어야 하므로 `% 256` 적용.

```python
def recurrence(n):
    if n == 0:
        return 3
    elif n == 1:
        return 5
    a, b = 3, 5
    for i in range(2, n + 1):
        a, b = b, (2 * b + 3 * a) % 256
    return b

# 암호화된 플래그 (0x4040에서 추출)
encrypted_flag = bytes.fromhex('7671c5a9e222d8b573f19228b2bf905a7677fca6b32190da6fb5cf38')

# 복호화
flag = ''
for i in range(len(encrypted_flag)):
    key = recurrence(i * i)
    decrypted = encrypted_flag[i] ^ key
    flag += chr(decrypted)

print(flag)
```

### 실행 결과

```
utflag{0pt1m1z3_ur_c0d3_l0l}
```

---

## 🚩 Flag

```
utflag{0pt1m1z3_ur_c0d3_l0l}