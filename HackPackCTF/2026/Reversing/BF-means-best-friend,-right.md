# [HackPackCTF 2026] BF-means-best-friend,-right

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 Brainfuck 언어로 작성된 프로그램(`main.bf`)을 분석하여 플래그를 찾는 리버싱 문제입니다. 일련의 메모리 조작을 통해 문자열을 구성하지만, 출력 명령(`.`)이나 입력 명령(`,`)이 없어 단순 실행만으로는 결과를 확인할 수 없습니다. 분석을 통해 메모리에 저장되는 문자열을 추출하고, 이를 플래그 형식으로 제출하면 됩니다.

- **Target:** `/sandbox/bins/main.bf` (Brainfuck 소스 코드)
- **핵심 포인트:** Brainfuck 코드의 메모리 조작 패턴 분석 및 문자열 복원

---

## 🔍 정찰

### 1) 파일 목록 확인 및 기본 정보 수집

먼저 주어진 디렉터리에 어떤 파일이 있는지 확인합니다. 문제 설명에서 `main.bf`가 언급되었지만, 추가 파일이 있을 수 있으므로 전체 목록을 확인합니다.

```bash
ls -la /sandbox/bins/
```

**관찰 결과:**
```
total 12
drwxr-xr-x 2 root root 4096 Mar 11 10:56 .
drwxr-xr-x 1 root root 4096 Mar 11 10:56 ..
-rwxr-xr-x 1 root root  253 Mar 11 10:56 main.bf
```

`main.bf` 하나만 존재하며, 크기는 253바이트입니다. 파일 확장자 `.bf`는 Brainfuck 코드임을 시사합니다.

### 2) Brainfuck 코드 내용 확인

파일의 내용을 확인하여 코드 구조를 분석합니다.

```bash
cat /sandbox/bins/main.bf
```

**관찰 결과:**
```
--[----->+<]>----[-<+>]+[--------->++<]>[-<+>]--[----->+<]>-----[-<+>]+[----->+++<]>++[-<+>]+[------->++<]>[-<+>]++[------>+<]>++[-<+>]--[----->+<]>----[-<+>]+[------->++<]>--[-<+>]--[----->+<]>-----[-<+>]+[--------->++<]>+[-<+>]--------[-->+++<]>[-<+>]
```

코드는 `+`, `-`, `<`, `>`, `[`, `]` 명령어로 구성되어 있으며, `,` (입력) 또는 `.` (출력) 명령어가 포함되어 있지 않습니다. 이는 프로그램이 외부 입력을 받지 않으며, 직접 출력도 하지 않는다는 의미입니다.

---

## 🧠 문제 분석

Brainfuck 프로그램은 메모리 테이프를 기반으로 동작하며, 주어진 코드는 반복적인 루프를 통해 특정 메모리 셀의 값을 조정합니다. 각 `[->+<]` 형태의 패턴은 일반적으로 값을 이동하거나 복사하는 데 사용됩니다.

핵심은 이 코드가 어떤 문자열을 메모리에 구성하고 있다는 점입니다. 코드를 세부적으로 분석하거나, Python 기반 Brainfuck 인터프리터를 사용해 실행 후 메모리 상태를 덤프하면, 어떤 값이 저장되는지 확인할 수 있습니다.

### Brainfuck 인터프리터를 통한 메모리 추적

다음은 코드를 실행하고 메모리 상태를 추적하는 Python 스크립트입니다.

```python
def brainfuck_interpret(code):
    memory = [0] * 30000
    pointer = 0
    pc = 0
    jump_map = {}
    stack = []

    # 괄호 매핑 생성
    for i, cmd in enumerate(code):
        if cmd == '[':
            stack.append(i)
        elif cmd == ']':
            if stack:
                start = stack.pop()
                jump_map[start] = i
                jump_map[i] = start

    while pc < len(code):
        cmd = code[pc]

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
                pc = jump_map[pc]
        elif cmd == ']':
            if memory[pointer] != 0:
                pc = jump_map[pc]

        pc += 1

    # 메모리 0~11번 셀 출력 (의심되는 문자열 영역)
    result = ''.join(chr(c) for c in memory[0:12] if 32 <= c <= 126)
    print("Memory [0:12]:", memory[0:12])
    print("Decoded string:", result)
    return result

# 코드 로드 및 실행
with open('/sandbox/bins/main.bf', 'r') as f:
    bf_code = f.read().strip()

flag_content = brainfuck_interpret(bf_code)
print("Potential flag content:", flag_content)
```

**실행 결과:**
```
Memory [0:12]: [98, 114, 97, 105, 110, 45, 98, 108, 97, 115, 116, 0]
Decoded string: brain-blast
Potential flag content: brain-blast
```

메모리 셀 0~10에 `b`, `r`, `a`, `i`, `n`, `-`, `b`, `l`, `a`, `s`, `t`가 저장되어 있으며, 이는 ASCII로 "brain-blast"를 의미합니다.

---

## 💥 익스플로잇

문제는 출력 명령이 없기 때문에, 단순히 프로그램을 실행해도 아무 출력이 없습니다. 그러나 메모리 분석을 통해 `"brain-blast"`가 명시적으로 구성되고 있음을 확인했습니다. 이는 플래그의 내용임을 강력히 시사합니다.

또한, 문제 제목인 **"BF-means-best-friend,-right"** 는 약간의 트릭을 포함하고 있지만, 분석 결과 실제 플래그는 코드 내에 하드코딩된 문자열을 기반으로 합니다. 다른 추측(예: `CTF{best-friend}`)은 오답이며, 실제 메모리에 저장된 값이 정답입니다.

따라서 플래그는 표준 형식인 `CTF{}`로 감싸서 제출합니다.

---

## 🚩 Flag

```
CTF{brain-blast}
```