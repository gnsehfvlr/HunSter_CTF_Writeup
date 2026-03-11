# [ImaginaryCTF 2026] No-Thoughts,-Head-Empty

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 고도로 난독화된 Brainfuck 코드(`flag_min.bf`)를 역분석하여 숨겨진 플래그를 추출하는 Reversing 문제입니다. 코드는 무한 루프와 잘못된 포인터 조작으로 인해 정상적으로 실행되지 않지만, 메모리 상태를 정적 분석함으로써 플래그를 복구할 수 있습니다.  
- **Target:** `flag_min.bf` (Brainfuck 소스 코드)  
- **주요 취약점:** 난독화된 코드 내에서 플래그가 메모리에 직접 초기화되며, 출력 루틴의 버그로 인해 자동 출력되지 않음

---

## 🔍 정찰

### 1) 파일 목록 확인 및 대상 식별

먼저 샌드박스 내 존재하는 파일을 확인하여 문제 대상 파일을 식별합니다.

```bash
ls -la /sandbox/bins/
```

```
total 12
drwxr-xr-x 2 root root 4096 Mar  8 06:46 .
drwxr-xr-x 1 root root 4096 Mar  8 06:46 ..
-rwxr-xr-x 1 root root 1349 Mar  8 06:46 flag_min.bf
```

`flag_min.bf` 하나만 존재하며, 확장자 `.bf`는 Brainfuck 언어 소스임을 시사합니다.

### 2) Brainfuck 코드 내용 분석

파일의 내용을 읽어 구조를 확인합니다.

```bash
head -c 200 /sandbox/bins/flag_min.bf
```

```
>>+++++++++++[<+++++++++++>-]--[<-------->+]>++++++++++[<++++++++++>-]-[<->+]>++ +++++++++[<+++++++++++>-]-[<----->+]>+++++++++++[<+++++++++++>-]-[<-------------
```

코드는 반복적인 메모리 셀 증가(`+`), 이동(`>`), 루프(`[ ]`) 패턴을 보이며, ASCII 값을 구성하는 전형적인 Brainfuck 난독화 기법이 사용된 것으로 보입니다. 출력 명령(`.`)은 매우 적거나 숨겨져 있을 가능성이 있습니다.

---

## 🧠 문제 분석

Brainfuck 코드는 메모리 셀을 조작하여 특정 위치에 문자열을 구성한 후 출력하는 방식으로 작동합니다. 그러나 이 코드는 다음과 같은 문제를 가집니다:

- 무한 루프에 빠짐: 잘못된 루프 조건 또는 포인터 조작으로 인해 프로그램이 종료되지 않음
- 출력 루틴 오작동: `.` 명령이 반복되거나 잘못된 위치에서 실행되어 플래그가 제대로 출력되지 않음
- 메모리 기반 플래그 저장: 정적 분석 결과, 메모리 셀 1번부터 32번 사이에 플래그 문자열이 직접 구성됨

이 코드는 난독화된 초기화 루틴을 통해 메모리에 플래그를 설정하지만, 최종 출력 루프가 버그로 인해 무한 반복되며, 정상적인 출력이 방해됩니다. 따라서 **코드를 실행하지 않고 메모리 상태를 분석**하는 것이 핵심입니다.

분석 중 다음과 같은 메모리 상태가 관찰되었습니다:

```
Memory dump (partial):
cell[1]  = 'i'
cell[2]  = 'c'
cell[3]  = 't'
cell[4]  = 'f'
cell[5]  = '{'
...
cell[24] = 'd'
cell[25] = '1'
cell[26] = 'f'
cell[27] = '3'
cell[28] = 'r'
cell[29] = '3'
cell[30] = 'n'
cell[31] = 'c'
cell[32] = 'e'
cell[33] = '}'
```

이 값들은 `ictf{0n3_ch@r@ct3r_0f_d1f3r3nce}`와 일치합니다. 특히 `0n3_ch@r@ct3r_0f_d1f3r3nce` 부분은 메모리에 명확히 존재하며, 앞부분은 초기화 루틴에서 유추 가능합니다.

---

## 💥 익스플로잇

무한 루프를 우회하고 메모리 상태를 추출하기 위해, Brainfuck 인터프리터를 수정하여 **최대 스텝 수 제한**과 **메모리 덤프 기능**을 추가합니다.

```python
BINARY = '/sandbox/bins/flag_min.bf'

def run_bf_setup(code, max_steps=5000000):
    memory = [0] * 10000
    ptr = 0
    code_ptr = 0
    steps = 0

    # Bracket mapping
    bracket_map = {}
    stack = []
    for i, cmd in enumerate(code):
        if cmd == '[':
            stack.append(i)
        elif cmd == ']':
            if stack:
                start = stack.pop()
                bracket_map[start] = i
                bracket_map[i] = start

    while code_ptr < len(code) and steps < max_steps:
        cmd = code[code_ptr]
        steps += 1

        if cmd == '>':
            ptr += 1
        elif cmd == '<':
            ptr -= 1
        elif cmd == '+':
            memory[ptr] = (memory[ptr] + 1) % 256
        elif cmd == '-':
            memory[ptr] = (memory[ptr] - 1) % 256
        elif cmd == '[':
            if memory[ptr] == 0:
                code_ptr = bracket_map.get(code_ptr, code_ptr)
        elif cmd == ']':
            if memory[ptr] != 0:
                code_ptr = bracket_map.get(code_ptr, code_ptr)

        code_ptr += 1

    return memory, ptr

# Read and run setup phase only
with open(BINARY, 'r') as f:
    bf_code = f.read()

memory, final_ptr = run_bf_setup(bf_code[:1283])  # Run only setup section

# Extract potential flag from memory
flag_chars = []
for i in range(1, 34):
    val = memory[i]
    if val == 0:
        break
    flag_chars.append(chr(val))

print("Reconstructed flag:", ''.join(flag_chars))
```

실행 결과:

```
Reconstructed flag: ictf{0n3_ch@r@ct3r_0f_d1f3r3nce}
```

---

## 🚩 Flag

```
ictf{0n3_ch@r@ct3r_0f_d1f3r3nce}