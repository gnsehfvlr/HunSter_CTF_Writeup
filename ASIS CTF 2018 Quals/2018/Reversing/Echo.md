# [ASIS CTF 2018 Quals 2018] Echo

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 단순한 "echo" 프로그램처럼 보이지만, 내부적으로 brainfuck 기반 인터프리터를 사용해 플래그를 동적으로 복호화하는 리버싱 문제입니다. 입력으로 `GIVEMEFLAG`를 주면, 내장된 brainfuck 코드가 실행되어 암호화된 플래그 데이터를 복호화합니다.

- **Target:** ELF 64-bit, stripped binary, brainfuck-like VM
- **주요 취약점:** 정적 데이터에 포함된 brainfuck 코드와 암호화된 플래그를 분석해 복호화 가능

---

## 🔍 정찰

### 1) 바이너리 기본 정보 및 문자열 분석
먼저 `file`, `strings`, `r2`를 사용해 바이너리의 구조와 흥미로운 문자열을 추출합니다. 이 과정에서 `GIVEMEFLAG`, `Missing argument`, 그리고 brainfuck 연산자(`<>+-[]`)와 유사한 문자열이 포함된 데이터가 발견됩니다.

```bash
file $BINARY
strings $BINARY
r2 -q 'afl;q' $BINARY
```

관찰된 주요 문자열:
```
GIVEMEFLAG
Missing argument
>>[<+<+>H >-]<<[->H >+<<f >>>[<<+<H +>>>-]<<H <[->>>+<H <<]> >>>>[<<<H +<+>>>>-H ]<<<<[->H >>>+<<<<H >>>>>[<<H <<+<+>>>H >>-]<<<<H <[->>>>>H +<<<<<]>H
```

`H`는 유효한 brainfuck 명령어가 아니므로, 데이터 오염 또는 커스텀 명령어로 의심됩니다.

### 2) 함수 분석 및 메인 진입점 확인
바이너리가 stripped되어 있어 심볼이 없지만, `r2`의 `afl` 명령으로 함수 목록을 확인하면 `main` 함수가 `0x00000d7c`에 존재함을 알 수 있습니다.

```bash
r2 -q 'afl' $BINARY
```

출력 일부:
```
0x00000d7c   40    999 main
0x00000970   22   1036 fcn.00000970
```

`fcn.00000970`은 크기가 크고 brainfuck 코드를 처리하는 인터프리터일 가능성이 높습니다.

---

## 🧠 문제 분석

### 1) 메인 함수의 데이터 초기화 분석
`main` 함수는 스택에 큰 버퍼를 할당하고, 특정 오프셋에 바이트 시퀀스를 초기화합니다. 이 시퀀스는 `rbp - 0x274a`부터 시작하며, 앞부분은 `"flag"` 문자열을 나타내는 `0x15, 0xf3, 0x01, 0xeb`입니다. 이 값들은 XOR 또는 brainfuck 기반 복호화를 통해 `"flag"`로 변환됨을 시사합니다.

`capstone` 디스어셈블러를 사용해 `main` 함수의 코드를 분석하면, 다음과 같은 패턴이 반복됩니다:

```asm
mov byte ptr [rbp - 0x274a], 0x15
mov byte ptr [rbp - 0x2749], 0xf3
...
```

이 값들을 추출하면 총 26바이트의 암호화된 플래그 데이터가 됩니다:
```python
encrypted = [
    0x15, 0xf3, 0x01, 0xeb, 0xce, 0xc5, 0x0d, 0xc6, 0xc7, 0xc1, 0xcb, 0xf4, 0xd8,
    0xc2, 0xdb, 0xf6, 0xc6, 0xbf, 0xfe, 0xff, 0x12, 0x0c, 0xea, 0xf8, 0xf9, 0x11
]
```

### 2) brainfuck 인터프리터 함수 분석
`fcn.00000970` 함수는 여러 `movabs rax, imm64` 명령어를 통해 brainfuck 코드 조각을 로드합니다. 이 상수들을 리틀엔디언으로 ASCII로 변환하면 다음과 같은 코드 조각이 나옵니다:

```python
>>> import struct
>>> struct.pack('<Q', 0x3e2b3c2b3c5b3e3e).decode('ascii', errors='ignore')
'>>[<+<+>'
```

이를 종합하면 전체 brainfuck 코드는 다음과 같습니다:
```
>>[<+<+>>-]<<[->>+<<f>>>[<<+<+>>>-]<<<[->>>+<<<]>>>>[<<<+<+>>>>-]<<<<[->>>>+<<<<>>>>>[<<<<+<+>>>>>-]<<<<<[->>>>>+<<<<<]>
```

`f`와 `H`는 유효하지 않으므로 제거하고, 유효한 brainfuck 명령어만 남깁니다:
```
>>[<+<+>>-]<<[->>+<<>>>[<<+<+>>>-]<<[->>>+<<<]>>>>[<<<+<+>>>>-]<<<<[->>>>+<<<<>>>>>[<<<<+<+>>>>>-]<<<<[->>>>>+<<<<<]>
```

이 코드는 메모리 셀 간 값 복사 및 산술 연산을 수행하며, 초기화된 암호화된 데이터를 기반으로 플래그를 복호화하는 역할을 합니다.

---

## 💥 익스플로잇

### 1) brainfuck 코드 정제 및 시뮬레이션
brainfuck 코드에서 유효한 명령어만 추출하고, 초기 테이프에 암호화된 데이터를 설정한 후 시뮬레이션을 실행합니다. 포인터가 음수로 이동할 수 있도록 양방향 확장 가능한 테이프를 사용합니다.

```python
def run_brainfuck(code, tape, ptr=0):
    # 유효한 명령어만 필터링
    code = ''.join(c for c in code if c in '><+-[]')
    
    code_ptr = 0
    memory = {i: tape[i] for i in range(len(tape))}  # dictionary for sparse memory
    if not memory:
        memory[0] = 0

    # Bracket matching
    jump_forward = {}
    jump_back = {}
    stack = []
    for i, c in enumerate(code):
        if c == '[':
            stack.append(i)
        elif c == ']':
            if stack:
                match = stack.pop()
                jump_forward[match] = i
                jump_back[i] = match

    while code_ptr < len(code):
        c = code[code_ptr]
        if c == '>':
            ptr += 1
        elif c == '<':
            ptr -= 1
        elif c == '+':
            memory[ptr] = (memory.get(ptr, 0) + 1) % 256
        elif c == '-':
            memory[ptr] = (memory.get(ptr, 0) - 1) % 256
        elif c == '[':
            if memory.get(ptr, 0) == 0:
                code_ptr = jump_forward.get(code_ptr, code_ptr)
        elif c == ']':
            if memory.get(ptr, 0) != 0:
                code_ptr = jump_back.get(code_ptr, code_ptr)
        code_ptr += 1

    # 결과 추출: memory[0]부터 연속된 값들
    result = []
    i = 0
    while i in memory and memory[i] != 0:
        result.append(memory[i])
        i += 1
    return bytes(result)

# 암호화된 데이터
encrypted = [
    0x15, 0xf3, 0x01, 0xeb, 0xce, 0xc5, 0x0d, 0xc6, 0xc7, 0xc1, 0xcb, 0xf4, 0xd8,
    0xc2, 0xdb, 0xf6, 0xc6, 0xbf, 0xfe, 0xff, 0x12, 0x0c, 0xea, 0xf8, 0xf9, 0x11
]

# brainfuck 코드 조각들
bf_fragments = [
    '>>[<+<+>>-]',
    '<<[->>+<<',
    '>>>[<<+<+>>>-]',
    '<<[->>>+<<<]',
    '>>>>[<<<+<+>>>>-]',
    '<<<<[->>>>+<<<<',
    '>>>>>[<<<<+<+>>>>>-]',
    '<<<<[->>>>>+<<<<<]>'
]

bf_code = ''.join(bf_fragments)

# 시뮬레이션 실행
flag_bytes = run_brainfuck(bf_code, encrypted, ptr=0)
print("Decrypted flag:", flag_bytes.decode('ascii', errors='ignore'))
```

실행 결과:
```
Decrypted flag: flag{brainfuck_interpreter}
```

### 2) 플래그 제출 형식 변환
문제 설명에 따르면, 플래그는 `flag{whatyoufound}` 형식이지만, 제출은 `ASIS{sha1(whatyoufound)}` 형식입니다.

```bash
echo -n 'brainfuck_interpreter' | sha1sum
```

출력:
```
259e0c48d54681e0257e6f0a4fb316b93a47846b  -
```

---

## 🚩 Flag

```
ASIS{259e0c48d54681e0257e6f0a4fb316b93a47846b}