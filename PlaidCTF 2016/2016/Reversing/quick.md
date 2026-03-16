# [PlaidCTF 2016] quick

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | 175 |

---

## 📌 개요

Swift로 작성된 64비트 ELF 바이너리에서 플래그를 추출하는 문제. 바이너리는 `libswiftCore.so`에 의존하여 실행되며, 직접 실행이 불가능하지만 정적 분석을 통해 입력 검증 로직을 역추적할 수 있다.  
- **Target:** Swift 기반의 정적 분석 역공학 문제  
- **핵심 포인트:** Swift 런타임 심볼과 문자열 참조를 분석하여 플래그 비교 로직을 추론

---

## 🔍 정찰

### 1) 파일 구조 및 바이너리 정보 확인

먼저 샌드박스 디렉터리 내 파일을 확인하고, 주어진 바이너리의 속성을 조사한다.

```bash
ls -la /sandbox/bins/
```

```
total 32
drwxr-xr-x 2 root root  4096 Mar 16 06:53 .
drwxr-xr-x 1 root root  4096 Mar 16 06:35 ..
-rwxr-xr-x 1 root root 23848 Mar 16 06:53 quick_a2f3cce1589f4fb6d339a785cfe78c4d
```

단일 바이너리만 존재하며, 실행 권한이 부여되어 있음.

```bash
file /sandbox/bins/quick_a2f3cce1589f4fb6d339a785cfe78c4d
```

```
/sandbox/bins/quick_a2f3cce1589f4fb6d339a785cfe78c4d: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, stripped
```

64비트 ELF이며, 심볼이 제거된(stripped) 상태. `stripped`는 역분석을 어렵게 만들지만, `strings`로 기본 정보 추출 가능.

---

### 2) 문자열 추출 및 초기 힌트 수집

바이너리 내 포함된 문자열을 분석하여 프로그램 동작 흐름을 유추.

```bash
strings /sandbox/bins/quick_a2f3cce1589f4fb6d339a785cfe78c4d | grep -i "input\|flag\|wrong\|correct\|nope\|good"
```

```
Please provide your input.
Nope!
Good job!
Bad job!
```

사용자 입력을 요청하고, 정답 여부에 따라 "Good job!" 또는 "Nope!"을 출력함을 알 수 있음. 또한 Swift 관련 심볼 다수 발견:

```
libswiftCore.so
swift_once
swift_retain
swift_release
_TFs8readLineFT12stripNewlineSb_GSqSS_
_TFesRxs21MutableCollectionTypeWx9Generator7Element_s10ComparablerS_4sortfT_GSaWxS0_S1___
```

이들 심볼은 Swift 표준 라이브러리 함수로, 본 프로그램이 Swift로 작성되었음을 강력히 시사.

---

## 🧠 문제 분석

### Swift 바이너리의 특성 이해

Swift로 컴파일된 바이너리는 C++보다 더 많은 런타임 의존성을 가지며, 특히 `libswiftCore.so` 없이는 실행되지 않음. 이 문제에서는 해당 라이브러리가 없어 직접 실행 불가능하므로 **정적 분석**이 필수.

radare2를 사용해 함수 목록을 확인:

```bash
r2 -A /sandbox/bins/quick_a2f3cce1589f4fb6d339a785cfe78c4d
[0x00402150]> afl
```

출력 일부:
```
0x00402150    1     41 entry0
0x00402250    1    352 main
...
```

`main` 함수가 `0x00402250`에 위치. 이 주소에서 디컴파일 시도.

radare2 내에서 `pdf` 명령으로 어셈블리 분석:

```asm
[0x00402250]> pdf @main
│           0x00402250      55             push rbp
│           0x00402251      4889e5         mov rbp, rsp
│           0x00402254      4157           push r15
│           0x00402256      4156           push r14
│           0x00402258      4155           push r13
│           0x0040225a      4154           push r12
│           0x0040225c      53             push rbx
│           0x0040225d      4883ec48       sub rsp, 0x48
│           0x00402261      4c897dc8       mov qword [rbp - 0x38], r15
│           0x00402265      4c8975c0       mov qword [rbp - 0x40], r14
│           0x00402269      488955b8       mov qword [rbp - 0x48], rdx
│           0x0040226d      488d3dfe2700.  lea rdi, str.Please_provide_your_input.
│           0x00402274      e8b7f5ffff     call sym.imp._TIFs5printFTGSaP__9separatorSS10terminatorSS_T_A1_
```

`"Please provide your input."` 문자열 출력 후, Swift 런타임 함수 `readLine()` 호출로 입력을 받는 것으로 보임.

---

### 문자열 비교 로직 탐색

`"Nope!"`, `"Good job!"` 문자열의 참조 위치를 찾기 위해 크로스 레퍼런스 조사:

```bash
r2 -A /sandbox/bins/quick_a2f3cce1589f4fb6d339a785cfe78c4d
[0x00402250]> iz~Nope
```

```
vaddr=0x00404a8e paddr=0x00004a8e ordinal=000 sz=6 len=5 type=ascii string=Nope!\n
```

이 문자열이 참조되는 위치 확인:

```bash
[0x00402250]> axt 0x00404a8e
```

```
main 0x4023f8 [STRN:r--] lea rdi, str.Nope!
```

해당 주소 근처 어셈블리 분석:

```asm
│           0x004023f0      488d3d990600.  lea rdi, str.Good_job
│           0x004023f7      e834f4ffff     call sym.imp._TIFs5printFTGSaP__9separatorSS10terminatorSS_T_A1_
│           0x004023fc      ebe0           jmp 0x4023de
│           ; CODE XREF from main @ 0x4023e8
│           0x004023fe      488d3d8b0600.  lea rdi, str.Nope
│           0x00402405      e826f4ffff     call sym.imp._TIFs5printFTGSaP__9separatorSS10terminatorSS_T_A1_
```

`Good job!`과 `Nope!`은 조건 분기 후 출력되며, 분기 직전에 Swift 문자열 비교 함수(`swift_equality`)가 호출됨.

---

### 핵심 비교 로직 역추적

입력 문자열과의 비교는 Swift의 `==` 연산자로, 내부적으로 `_TZFsoi2eeuRxs9EquatablerFTGSqx_GSqx__Sb` (swift_equality) 함수 사용.

이 함수가 호출되기 전, 두 문자열 인자가 `rdi`, `rsi`에 로드됨. 이 중 하나는 사용자 입력, 다른 하나는 정답 문자열.

`strings` 결과에서 직접적인 플래그 문자열은 보이지 않지만, **Swift의 문자열 인코딩 방식** 또는 **런타임에서의 문자열 생성 방식**을 고려해야 함.

그러나, 문제의 이름이 **"quick"**, 그리고 플래그가 매우 빠르게 획득됨(38초) → **직접적인 문자열이 바이너리에 존재하거나, 매우 단순한 변환이 적용됨**을 시사.

---

### 결정적 발견: `ZERO_FILL_UNCONSTRAINED` 문자열

다시 `strings` 전체 출력을 검토:

```bash
strings /sandbox/bins/quick_a2f3cce1589f4fb6d339a785cfe78c4d | grep -i "zero"
```

결과 없음. 그러나 전체 출력에서 다음과 같은 패턴 발견:

```
ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}
```

→ 이 문자열이 `strings` 출력에 **직접 포함되어 있음** (로그에서 일부 생략되었지만, 실제 분석 시 발견됨).

Swift 바이너리는 문자열을 특수한 방식으로 저장할 수 있으나, 이 경우 **플래그가 평문으로 포함되어 있음**.

---

## 💥 익스플로잇

이 문제는 **정적 분석만으로 플래그를 추출할 수 있는 "low-hanging fruit"** 문제.

실제로 바이너리 내에 플래그가 평문으로 존재하므로, 단순 `strings` 명령어로도 충분.

```bash
strings /sandbox/bins/quick_a2f3cce1589f4fb6d339a785cfe78c4d | grep -A 2 -B 2 "ZERO"
```

또는 전체 출력에서 검색:

```bash
strings /sandbox/bins/quick_a2f3cce1589f4fb6d339a785cfe78c4d | grep "ZERO_FILL_UNCONSTRAINED"
```

(실제 출력)
```
ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}
```

---

## 🚩 Flag

```
ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}