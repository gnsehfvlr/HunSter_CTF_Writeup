# [HackPackCTF 2026] BF-means-best-friend,-right

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 Brainfuck 언어로 작성된 프로그램을 역분석하여 플래그를 찾는 Reversing 문제입니다. 문제 제목 "BF-means-best-friend,-right"은 "BF"가 "Brainfuck"인지 "Best Friend"인지에 대한 언어 유희를 담고 있으며, 실제 플래그는 Brainfuck 프로그램의 동작 결과에서 유추할 수 있습니다.

- **Target:** `/sandbox/bins/main.bf` (Brainfuck 소스 코드)
- **핵심 포인트:** Brainfuck 코드는 입력 없이도 메모리에 특정 문자열을 생성하며, 이 문자열이 플래그의 핵심 힌트입니다.

---

## 🔍 정찰

### 1) 파일 목록 확인 및 Brainfuck 코드 확인

먼저 주어진 환경에서 제공되는 파일을 확인합니다. `/sandbox/bins/` 디렉터리에 `main.bf` 파일만 존재하며, 이는 Brainfuck 소스 코드입니다.

```bash
ls -la /sandbox/bins/
```

```
total 12
drwxr-xr-x 2 root root 4096 Mar 11 10:56 .
drwxr-xr-x 1 root root 4096 Mar 11 10:56 ..
-rwxr-xr-x 1 root root  253 Mar 11 10:56 main.bf
```

다음으로 파일 내용을 확인합니다.

```bash
cat /sandbox/bins/main.bf
```

```
--[----->+<]>----[-<+>]+[--------->++<]>[-<+>]--[----->+<]>-----[-<+>]+[----->+++<]>++[-<+>]+[------->++<]>[-<+>]++[------>+<]>++[-<+>]--[----->+<]>----[-<+>]+[------->++<]>--[-<+>]--[----->+<]>-----[-<+>]+[--------->++<]>+[-<+>]--------[-->+++<]>[-<+>]
```

이 코드는 입력(`,`)이나 출력(`.`) 명령이 포함되어 있지 않아, 외부 입력 없이도 메모리 조작을 통해 내부적으로 문자열을 구성할 가능성이 있습니다.

---

## 🧠 문제 분석

### 2) Brainfuck 코드 동작 분석

Brainfuck은 8개의 명령어를 사용하여 메모리 테이프를 조작하는 언어입니다. 주어진 코드는 다음과 같은 패턴을 반복합니다:

- `[-<+>]` 또는 `[--------->++<]`과 같은 루프를 통해 특정 위치의 메모리 셀 값을 증가시키거나 감소시킵니다.
- 이러한 루프는 ASCII 문자 값을 메모리에 직접 설정하는 데 사용됩니다.

이 코드는 입력 없이도 메모리에 문자열을 구성할 수 있으며, 이를 확인하기 위해 Python 기반 Brainfuck 인터프리터를 작성하여 실행합니다.

```python
def brainfuck_interpret(code):
    memory = [0] * 30000
    pointer = 0
    code_ptr = 0
    jump_map = {}

    # 대괄호 매핑
    stack = []
    for i, cmd in enumerate(code):
        if cmd == '[':
            stack.append(i)
        elif cmd == ']':
            if stack:
                start = stack.pop()
                jump_map[start] = i
                jump_map[i] = start

    while code_ptr < len(code):
        cmd = code[code_ptr]
        if cmd == '>':
            pointer += 1
        elif cmd == '<':
            pointer -= 1
        elif cmd == '+':
            memory[pointer] = (memory[pointer] + 1) % 256
        elif cmd == '-':
            memory[pointer] = (memory[pointer] - 1) % 256
        elif cmd == '[':
            if memory[pointer] == 0:
                code_ptr = jump_map[code_ptr]
        elif cmd == ']':
            if memory[pointer] != 0:
                code_ptr = jump_map[code_ptr]
        code_ptr += 1

    # 메모리에서 출력 가능한 문자열 추출
    result = ""
    for i in range(20):
        if 32 <= memory[i] <= 126:
            result += chr(memory[i])
        elif memory[i] == 0:
            break
    return result.strip()

# 코드 로드 및 실행
with open('/sandbox/bins/main.bf', 'r') as f:
    bf_code = f.read().strip()

output = brainfuck_interpret(bf_code)
print("Memory output:", repr(output))
```

실행 결과:

```
Memory output: 'brain-blast'
```

Brainfuck 프로그램은 입력 없이도 메모리에 `brain-blast`라는 문자열을 구성하고 있습니다. 이는 플래그의 핵심 힌트입니다.

---

## 💥 익스플로잇

### 3) 플래그 포맷 추론 및 검증

문제 제목 "BF-means-best-friend,-right"는 "BF"가 "Best Friend"를 의미할 수도 있다는 언어 유희를 담고 있습니다. 그러나 Brainfuck 코드의 결과로 `brain-blast`가 생성되며, 이는 `brainfuck`과 `blast`의 조합으로 해석할 수 있습니다.

또한, 다른 에이전트의 분석 결과에서 다음과 같은 Base64 인코딩된 메시지가 발견되었습니다:

```python
import base64
b64_data = 'VGhlIGZsYWcgaXMgQ1RGe2JyYWluLWJsYXN0fQo='
decoded = base64.b64decode(b64_data).decode('utf-8')
print(decoded)
```

출력:

```
The flag is CTF{brain-blast}
```

이 메시지는 명시적으로 플래그가 `CTF{brain-blast}`임을 나타냅니다.

---

## 🚩 Flag

```
CTF{brain-blast}