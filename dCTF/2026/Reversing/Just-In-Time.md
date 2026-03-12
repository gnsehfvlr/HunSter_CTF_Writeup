# [dCTF 2026] Just-In-Time

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

`justintime` 바이너리는 단순한 XOR 기반 복호화 루틴을 사용하여 플래그를 런타임에 복구하는 문제입니다. 이름에서 암시하는 JIT(Just-In-Time)는 실제 코드 생성이 아니라, 바이너리 이름을 키로 사용해 플래그를 복호화하는 방식의 낚시 요소입니다.

- **Target:** ELF 64-bit LSB pie executable, stripped
- **핵심 포인트:** 플래그는 바이너리 내에 XOR로 암호화되어 있으며, 복호화 키는 `argv[0]` (실행 파일 이름)에서 유래

---

## 🔍 정찰

### 1) 파일 정보 및 문자열 분석

바이너리의 기본 정보를 확인하고, 내부 문자열을 추출하여 힌트를 수집합니다.

```bash
file /sandbox/bins/justintime
```

```
/sandbox/bins/justintime: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=62f8ef3708bd28228989d5aef4a9da8d6c16ca6a, for GNU/Linux 3.2.0, stripped
```

문자열 분석을 통해 복호화 완료 메시지와 포맷 문자열을 확인합니다.

```bash
strings /sandbox/bins/justintime | grep -i 'flag\|wrong\|correct\|enter\|secret\|key\|decrypt'
```

```
Decryption finished.
%02x
```

`Decryption finished.` 메시지는 플래그 복호화가 내부에서 수행됨을 시사합니다.

---

### 2) 동작 테스트

임의의 입력을 주어도 출력이 동일한지 확인합니다.

```bash
echo "test_input" | /sandbox/bins/justintime
```

```
Decryption finished.
```

입력과 관계없이 동일한 출력이 나오므로, 사용자 입력은 무시되며 플래그는 내부적으로만 처리됩니다.

---

## 🧠 문제 분석

### 1) Radare2로 함수 분석

Radare2를 사용해 함수 목록을 확인하고, 메모리 관련 시스템 콜이 있는지 조사합니다.

```bash
r2 -A /sandbox/bins/justintime
[0x00001060]> afl~mprotect;mmap;malloc
```

결과에서 `mprotect`, `mmap`은 없으나 `malloc`이 사용됨을 확인합니다. 이는 동적 메모리 할당은 있으나 JIT 코드 생성은 아님을 시사합니다.

### 2) main 함수 디컴파일

Radare2의 `pdc` 명령어로 `main` 함수를 디컴파일합니다.

```c
int main(int argc, char **argv) {
    void *buf = malloc(0x40);
    // 스택에 하드코딩된 암호화된 플래그 조각들
    uint64_t enc1 = 0x486765792038261b;
    uint64_t enc2 = 0x754b623167242872;
    uint64_t enc3 = 0x747d4e603566227b;
    uint64_t enc4 = 0x252f764e31333323;
    uint32_t enc5 = 0x46313160;
    uint16_t enc6 = 0x3123;

    memcpy(buf, &enc1, 8);
    memcpy((char*)buf + 8, &enc2, 8);
    memcpy((char*)buf + 16, &enc3, 8);
    memcpy((char*)buf + 24, &enc4, 8);
    memcpy((char*)buf + 32, &enc5, 4);
    memcpy((char*)buf + 36, &enc6, 2);

    // 키 생성 함수 호출
    char *key = fcn.0000126a(argv[0]); // argv[0]에서 키 생성

    // 복호화 함수 호출
    fcn.000011c5(buf, key, 0x27); // 39바이트 복호화

    printf("Decryption finished.\n");
    free(buf);
    return 0;
}
```

### 3) 키 생성 함수 분석 (`fcn.0000126a`)

이 함수는 `argv[0]` (실행 파일 이름)을 파일로 열어 처음 8바이트를 읽고, `NULL` 문자로 종료된 7바이트 키를 반환합니다.

```c
char *fcn.0000126a(char *filename) {
    FILE *f = fopen(filename, "rb");
    if (!f) return NULL;
    char *key = malloc(8);
    fread(key, 1, 8, f);
    key[7] = 0; // null-terminate
    fclose(f);
    return key;
}
```

즉, **실행 파일 이름과 동일한 이름의 파일이 존재해야 하며**, 그 파일의 처음 7바이트가 복호화 키로 사용됩니다.

---

## 💥 익스플로잇

복호화 과정은 다음과 같습니다:

1. 바이너리 이름이 `justintime`이므로, `justintime`이라는 이름의 파일을 생성
2. 이 파일의 처음 7바이트가 키로 사용됨
3. 하드코딩된 암호화된 플래그 바이트들을 이 키로 XOR 복호화

다음 파이썬 스크립트로 복호화를 수행할 수 있습니다.

```python
import struct

# 암호화된 플래그 조각들 (main 함수 내 스택 변수에서 추출)
encrypted = b''
encrypted += struct.pack('<Q', 0x486765792038261b)
encrypted += struct.pack('<Q', 0x754b623167242872)
encrypted += struct.pack('<Q', 0x747d4e603566227b)
encrypted += struct.pack('<Q', 0x252f764e31333323)
encrypted += struct.pack('<I', 0x46313160)
encrypted += struct.pack('<H', 0x3123)
encrypted = encrypted[:0x27]  # 총 39바이트

# 키: 바이너리 이름과 동일한 파일에서 처음 7바이트 읽기
with open('/sandbox/bins/justintime', 'rb') as f:
    key = f.read(7)  # 바이너리 자체의 첫 7바이트를 키로 사용

# XOR 복호화 (반복 키)
flag = b''
for i in range(len(encrypted)):
    flag += bytes([encrypted[i] ^ key[i % len(key)]])

print(f"Decrypted flag: {flag.decode()}")
```

실행 결과:

```
Decrypted flag: dctf{df77dbe0c407dd4a188e12013ccb009f}
```

---

## 🚩 Flag

```
dctf{df77dbe0c407dd4a188e12013ccb009f}