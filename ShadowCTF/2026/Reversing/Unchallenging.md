# [ShadowCTF 2026] Unchallenging

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy (로그 내용으로 추정) |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 단순한 리버싱 기반의 CTF 문제로, 바이너리 내부에 하드코딩된 문자열을 분석하고 입력을 통해 플래그를 유도하는 것이 핵심이다.  
- **Target:** ELF 64비트 바이너리 (`unchallenging`)  
- **주요 취약점:** `gets`를 통한 입력 처리 및 `strcmp`를 이용한 단순 문자열 비교 로직, 플래그는 입력 값의 의미적 해석을 통해 도출됨.

---

## 🔍 정찰

### 1) 바이너리 기본 정보 확인

문제 바이너리에 대해 기본적인 정찰을 수행하여 파일 형식, 포함된 문자열, 함수 목록을 확인한다. 이는 리버싱의 첫 번째 단계로, 중요한 힌트를 제공할 수 있다.

```bash
file unchallenging
strings -n 5 unchallenging
r2 -q -c 'afl' unchallenging
```

**관찰 결과:**
- ELF 64비트, PIE 기반 실행 파일로, 동적 링크됨.
- 중요한 문자열 발견:
  - `What is the password?`
  - `op3n_se5ame`
  - `{Ar@b1an_night5}`
  - `Wrong!!`
- `strcmp`, `gets`, `puts` 등의 함수 사용 확인.

이 중 `op3n_se5ame`와 `{Ar@b1an_night5}`는 플래그와 관련된 중요한 후보로 보인다.

---

### 2) main 함수 디컴파일 분석

Radare2의 디컴파일 기능을 사용해 `main` 함수의 로직을 분석한다.

```bash
r2 -A -q -c 'pdf @ main' unchallenging
# 또는 GUI 도구에서 pdg (pseudocode decompiler) 사용
```

**디컴파일된 코드 (간략화):**
```c
ulong main(int argc, char **argv) {
    char input[100];
    puts("What is the password?");
    gets(input);
    if (strcmp(input, "op3n_se5ame") == 0) {
        puts("{Ar@b1an_night5}");
    } else {
        puts("Wrong!!");
    }
    return 0;
}
```

프로그램은 사용자 입력을 `gets`로 읽은 후, `"op3n_se5ame"`과 비교하고 일치하면 `{Ar@b1an_night5}`를 출력한다.

---

## 🧠 문제 분석

이 바이너리는 보안적으로 취약한 함수인 `gets`를 사용하고 있지만, 문제의 핵심은 **버퍼 오버플로우 익스플로잇이 아니라 로직 분석과 문자열 해석**에 있다.

- `strcmp`를 통해 입력이 `"op3n_se5ame"`과 정확히 일치하는지 검사.
- 일치 시 `{Ar@b1an_night5}` 출력 → 이 문자열은 `Arabian Nights` (천일야화)를 의미하는 레트스피크(leet-speak) 형태.
- `op3n_se5ame` 역시 `open_sesame`의 레트스피크 변형임을 알 수 있음:
  - `3` → `e`
  - `5` → `s`

따라서 플래그는 단순 출력 문자열이 아니라, **의미적으로 올바른 형태로 복원된 문구**일 가능성이 높다.

---

## 💥 익스플로잇

### 1) 입력 테스트 및 출력 확인

`pwntools`를 사용해 바이너리에 정답으로 추정되는 입력을 제공하고 출력을 확인한다.

```python
from pwn import *

p = process("./unchallenging")
p.sendline(b"op3n_se5ame")
output = p.recvall().decode(errors='ignore')
print(repr(output))
```

**출력 결과:**
```
'What is the password?\n{Ar@b1an_night5}\n'
```

정확히 `{Ar@b1an_night5}`가 출력됨. 하지만 이 문자열을 그대로 플래그로 제출하면 형식 오류 발생.

### 2) 레트스피크 디코딩 시도

`op3n_se5ame` → `open_sesame`로 복원됨을 확인.

```python
s = 'op3n_se5ame'
s = s.replace('3', 'e').replace('5', 's')
print(s)  # 출력: open_sesame
```

동일한 방식으로 출력 문자열도 디코딩:

```python
s = '{Ar@b1an_night5}'
s = s.replace('@', 'a').replace('1', 'i').replace('5', 's')
print(s)  # 출력: {Arabian_nights}
```

하지만 `{Arabian_nights}`도 플래그로 인정되지 않음.

### 3) 플래그 형식 재검토 및 최종 추론

모든 시도가 실패한 후, **플래그는 출력이 아니라 입력의 의미**에 있다는 점에 주목.

- `op3n_se5ame` → `open_sesame`
- `open_sesame`는 천일야화에서 "문이 열리라"는 마법의 주문.
- 문제명 `Unchallenging`은 "간단하다"는 의미로, 플래그도 직관적일 것임을 시사.

또한, **플래그 포맷은 `CTF{...}`임이 일반적**이므로, `open_sesame`을 이 형식에 맞게 감싸면 정답일 가능성이 높다.

최종적으로, `CTF{open_sesame}`를 제출하여 성공.

---

## 🚩 Flag

```
CTF{open_sesame}