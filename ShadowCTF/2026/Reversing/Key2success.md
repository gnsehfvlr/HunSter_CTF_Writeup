# [ShadowCTF 2026] Key2success

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 주어진 ELF 바이너리를 역분석하여 올바른 입력을 찾아 플래그를 출력하게 만드는 전형적인 리버싱 문제입니다.  
프로그램은 사용자 입력을 받아 사전 정의된 키와 비교하며, 일치할 경우 `print_flag` 함수를 호출하여 플래그를 출력합니다.  
- **Target:** ELF 64-bit, PIE 활성화, 동적 링크, not stripped  
- **주요 취약점:** 하드코딩된 키 문자열 비교 (strcmp 기반), 초기화 함수를 통한 문자열 조작 없음

---

## 🔍 정찰

### 1) 초기 파일 분석 및 바이너리 특성 확인

먼저, 제공된 바이너리의 기본 정보를 확인하고, 어떤 라이브러리를 사용하는지, 어떤 문자열이 포함되어 있는지 조사합니다.

```bash
ls -la /sandbox/bins/ && file $BINARY && strings $BINARY | grep -v '/lib/' | grep -v '/usr/' | head -n 20
```

**관찰 결과:**
```
total 28
drwxr-xr-x 2 root root  4096 Mar 10 18:52 .
drwxr-xr-x 1 root root  4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root 17480 Mar 10 18:52 key2sucess
/sandbox/bins/key2sucess: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c575537266d34304437cd8c94763191d1918f457, for GNU/Linux 3.2.0, not stripped
uSrf
mgUa
puts
putchar
stdin
printf
fgets
strlen
stdout
malloc
usleep
__cxa_finalize
setbuf
strcmp
__libc_start_main
free
libc.so.6
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
```

- 바이너리는 PIE (Position Independent Executable)이며, `strcmp`, `fgets`, `puts` 등의 함수를 사용하고 있음
- `not stripped`이므로 심볼 정보가 존재함
- `strings` 출력에서 `strcmp`와 `fgets` 사용이 확인되어 입력 비교 로직이 존재할 가능성 높음

---

### 2) main 함수 분석을 통한 흐름 파악

`main` 함수를 디컴파일하여 프로그램의 실행 흐름을 확인합니다.

```bash
decompile {"function": "main"}
```

**디컴파일 결과:**
```c
bool main(int argc, char **argv, char **envp) {
    int iVar1;
    uint in_RDI;
    
    sym.print_intro(CONCAT44(in_RDI, argc));
    iVar1 = sym.check_password();
    if (iVar1 == 0) {
        sym.slow_type(0x20a7);  // "key.\n"
    } else {
        sym.slow_type(0x2088);  // "key:\n"
        sym.print_flag();
    }
    return iVar1 == 0;
}
```

- `check_password()`의 반환값이 0이면 실패 메시지 출력
- **반환값이 0이 아니면 `print_flag()` 호출** → 성공 조건은 `check_password()`가 **비제로**를 반환하는 것
- 즉, `check_password()`가 **1을 반환**해야 플래그 출력

---

## 🧠 문제 분석

### 1) check_password 함수 분석

`check_password` 함수의 분석을 시도했으나, 초기 디컴파일 시도에서 주소 오류 발생.  
radare2로 함수 목록을 확인하여 정확한 주소를 파악합니다.

```bash
r2 -q "aaa; afl~check"
```

**출력:**
```
0x000012dd    4    125 sym.check_password
```

- `sym.check_password`는 `0x12dd`에 위치하며, 크기 125바이트

### 2) 정확한 파일 오프셋 계산

PIE 바이너리이므로 가상 주소(VA)와 파일 오프셋(FO)을 정확히 매핑해야 함.  
`lief`를 사용해 `.text` 섹션의 위치를 확인:

```python
import lief
BINARY = '/sandbox/bins/key2sucess'
binary = lief.parse(BINARY)
text_section = binary.get_section('.text')
symbol = binary.get_symbol('check_password')
func_offset = symbol.value - text_section.virtual_address + text_section.file_offset
print(f"Function file offset: {func_offset:#x}")
```

**출력:**
```
.text section: 0x10f0 @ 0x10f0, size=0x331
check_password symbol: 0x12dd (size=125)
Function file offset: 0x12dd
```

- `.text` 섹션이 VA 0x10f0, 파일 오프셋 0x10f0 → **ASLR 없이 직접 읽을 수 있음**

### 3) check_password 어셈블리 분석

capstone를 사용해 `0x12dd`부터 125바이트를 디스어셈블:

```python
from capstone import *
BINARY = '/sandbox/bins/key2sucess'
with open(BINARY, 'rb') as f:
    f.seek(0x12dd)
    code = f.read(125)
md = Cs(CS_ARCH_X86, CS_MODE_64)
for ins in md.disasm(code, 0x12dd):
    print(f"0x{ins.address:x}: {ins.mnemonic} {ins.op_str}")
```

**핵심 분석 결과:**
- `puts@plt` 호출 → `"key:\n"` 출력
- `fgets`로 입력을 스택 버퍼에 저장
- `strlen` 후 마지막 바이트 제거 (개행 제거)
- `strcmp` 호출: 입력과 `[rip + 0x2d4a]` 주소의 문자열 비교
  - `0x12dd + 0x2d4a = 0x4080` → 비교 대상 주소: `0x4080`

### 4) 비교 문자열 추출

`0x4080`은 `.data` 섹션에 위치. `lief`로 섹션 매핑:

```python
for section in binary.sections:
    start = section.virtual_address
    end = start + section.size
    if start <= 0x4080 < end:
        offset = 0x4080 - start + section.file_offset
        with open(BINARY, 'rb') as f:
            f.seek(offset)
            data = f.read(32)
            print(data.split(b'\x00')[0].decode())
```

**출력:**
```
Constant_learning_is_the_key
```

- `0x4080`에는 문자열 `"Constant_learning_is_the_key"`가 저장됨
- 이 문자열은 `obj.the_password` 심볼로 확인 가능

### 5) 초기화 함수 확인

PIE 바이너리에서 `.data`가 런타임에 수정될 수 있으므로, 초기화 함수 확인:

```bash
r2 -q "afl~init"
decompile {"function": "entry.init0"}
decompile {"function": "sym._init"}
```

- `entry.init0`, `sym._init` 모두 비어 있음 → **문자열은 런타임에 수정되지 않음**

---

## 💥 익스플로잇

### 1) 입력 테스트

`check_password`는 입력과 `"Constant_learning_is_the_key"`를 `strcmp`로 비교하며, 일치하면 1을 반환 → 플래그 출력

```bash
echo -n "Constant_learning_is_the_key" | /sandbox/bins/key2sucess
```

또는 Python + pwntools로 테스트:

```python
from pwn import *
p = process('/sandbox/bins/key2sucess')
p.recvuntil(b'> ')
p.sendline(b'Constant_learning_is_the_key')
print(p.recvall().decode())
p.close()
```

**실행 결과:**
```
key:
nflag{Never_stop_learning}
```

- 입력이 정확히 일치하면 `key:\n` 출력 후 플래그 출력

---

## 🚩 Flag

```
nflag{Never_stop_learning}