# [HackPackCTF 2026] BF-means-best-friend,-right

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 Brainfuck 언어로 작성된 `main.bf` 파일을 역분석하여 숨겨진 플래그를 찾는 Reversing 문제이다.  
Brainfuck 프로그램은 입력이나 출력 없이 메모리에 특정 문자열을 직접 구성하며, 플래그는 해당 문자열을 기반으로 유추할 수 있다.

- **Target:** `main.bf` (Brainfuck 소스코드)
- **주요 취약점:** 하드코딩된 문자열을 통한 플래그 노출

---

## 🔍 정찰

### 1) 파일 구조 확인

먼저 제공된 디렉터리 내 파일을 확인하여 분석 대상을 파악한다.

```bash
ls -la /sandbox/bins/
```

```
total 12
drwxr-xr-x 2 root root 4096 Mar 10 15:31 .
drwxr-xr-x 1 root root 4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root  253 Mar 10 15:31 main.bf
```

분석 결과, 유일한 파일은 `main.bf`이며, Brainfuck 소스코드임을 알 수 있다.

---

### 2) Brainfuck 코드 내용 분석

다음으로 `main.bf` 파일의 내용을 확인하여 프로그램의 동작을 파악한다.

```bash
cat /sandbox/bins/main.bf
```

```
--[----->+<]>----[-<+>]+[--------->++<]>[-<+>]--[----->+<]>-----[-<+>]+[----->+++<]>++[-<+>]+[------->++<]>[-<+>]++[------>+<]>++[-<+>]--[----->+<]>----[-<+>]+[------->++<]>--[-<+>]--[----->+<]>-----[-<+>]+[--------->++<]>+[-<+>]--------[-->+++<]>[-<+>]
```

코드는 순수한 메모리 조작 명령어(`+`, `-`, `<`, `>`, `[`, `]`)로 구성되어 있으며, `.`(출력) 또는 `,`(입력) 명령어가 포함되어 있지 않다. 이는 프로그램이 **출력을 하지 않고**, 단순히 메모리 상태를 조작한다는 것을 의미한다.

---

## 🧠 문제 분석

Brainfuck 프로그램은 메모리 셀을 조작하여 특정 ASCII 값을 구성하는 방식으로 문자열을 생성한다.  
주어진 코드는 복잡한 루프와 산술 연산을 통해 메모리 셀에 값을 설정하지만, 실제로는 **문자열 "brain-blast"를 메모리에 하드코딩**하고 있다.

이를 확인하기 위해 Brainfuck 인터프리터를 사용하여 코드를 실행하고 메모리 상태를 분석한다.

```python
def interpret_bf(code):
    memory = [0] * 30000
    pointer = 0
    i = 0
    while i < len(code):
        command = code[i]
        if command == '>':
            pointer += 1
        elif command == '<':
            pointer -= 1
        elif command == '+':
            memory[pointer] = (memory[pointer] + 1) % 256
        elif command == '-':
            memory[pointer] = (memory[pointer] - 1) % 256
        elif command == '[':
            if not memory[pointer]:
                loop_depth = 1
                while loop_depth > 0:
                    i += 1
                    if code[i] == '[':
                        loop_depth += 1
                    elif code[i] == ']':
                        loop_depth -= 1
        elif command == ']':
            if memory[pointer]:
                loop_depth = 1
                while loop_depth > 0:
                    i -= 1
                    if code[i] == ']':
                        loop_depth += 1
                    elif code[i] == '[':
                        loop_depth -= 1
                i -= 1
        i += 1
    return memory

# 코드 실행
code = "--[----->+<]>----[-<+>]+[--------->++<]>[-<+>]--[----->+<]>-----[-<+>]+[----->+++<]>++[-<+>]+[------->++<]>[-<+>]++[------>+<]>++[-<+>]--[----->+<]>----[-<+>]+[------->++<]>--[-<+>]--[----->+<]>-----[-<+>]+[--------->++<]>+[-<+>]--------[-->+++<]>[-<+>]"
memory = interpret_bf(code)

# 메모리에서 출력 가능한 문자열 추출
output_str = ""
for i in range(20):
    if memory[i] != 0:
        output_str += chr(memory[i])
    else:
        break
print("Memory string:", output_str)
```

**실행 결과:**
```
Memory string: brain-blast
```

메모리의 처음 몇 바이트에 `brain-blast`라는 문자열이 정확히 구성되어 있음을 확인할 수 있다.  
또한, 코드 내에 `.` 또는 `,` 명령어가 없어 **어떤 입력도 받지 않으며**, **어떤 출력도 수행하지 않는다**.  
따라서 이 문자열 자체가 플래그의 단서임을 유추할 수 있다.

---

## 💥 익스플로잇

이 문제는 전통적인 익스플로잇보다는 **역분석을 통한 문자열 추출**이 핵심이다.  
Brainfuck 코드를 실행하여 메모리에 생성된 문자열을 확인하면, 플래그는 단순히 이 문자열을 플래그 포맷으로 감싼 형태임을 추정할 수 있다.

즉, `brain-blast` → `flag{brain-blast}`

이를 검증하기 위해 플래그를 제출하면 정답으로 인정된다.

---

## 🚩 Flag

```
flag{brain-blast}
```