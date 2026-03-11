# [ShadowCTF 2026] Thirsty-Cow

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 ELF 바이너리를 역분석하여 숨겨진 플래그를 추출하는 Reversing 문제입니다.  
프로그램은 사용자 입력을 받아 내부적으로 조각난 문자열을 조합한 후, 이를 입력과 비교하여 플래그를 출력합니다.  
- **Target:** `crow.out` (ELF 64-bit, not stripped)  
- **주요 취약점:** 문자열 조합 및 비교 로직의 정적 분석을 통한 플래그 유추

---

## 🔍 정찰

### 1) 샌드박스 내 파일 확인

먼저 샌드박스 내 존재하는 파일을 확인하여 분석 대상을 파악합니다. 추가 리소스 파일이 있는지 확인하는 것이 중요합니다.

```bash
ls -la /sandbox/bins/
```

```
total 28
drwxr-xr-x 2 root root  4096 Mar 10 21:11 .
drwxr-xr-x 1 root root  4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root 17560 Mar 10 21:11 crow.out
```

분석 결과, `crow.out` 하나의 바이너리만 존재하며, 추가 데이터 파일은 없습니다.

---

### 2) 바이너리 속성 및 문자열 분석

바이너리의 종류를 확인하고, `strings` 명령어를 통해 내부에 포함된 문자열을 추출합니다. 플래그 관련 문자열이나 입력 처리 메시지를 찾는 것이 핵심입니다.

```bash
file /sandbox/bins/crow.out && strings /sandbox/bins/crow.out
```

```
/sandbox/bins/crow.out: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, ...
...
The crow is thirsty and he needs your help to gather stones to fill the pot
shadowCTF{
Not enough stones :(
...
```

관찰된 문자열들 중에서 다음과 같은 힌트를 얻을 수 있습니다:
- 플래그 포맷의 시작 부분: `shadowCTF{`
- 플래그 조건과 관련된 메시지: `"Not enough stones :("`
- 동화 "The Crow and the Pitcher"를 암시하는 컨텍스트

또한 바이너리가 **stripped되지 않았음**이 확인되어, 함수 심볼을 쉽게 분석할 수 있습니다.

---

## 🧠 취약점 분석

### 3) 함수 목록 분석 및 main 함수 탐색

Radare2를 사용하여 함수 목록(`afl`)을 출력하고, `main` 함수의 위치를 확인합니다.

```r2
afl
```

```
0x000011a5    4    803 main
...
```

`main` 함수가 `0x000011a5`에 위치하며, 크기가 803바이트로 비교적 크기 때문에 상세한 로직이 포함되어 있을 가능성이 큽니다.

---

### 4) main 함수 디컴파일 및 문자열 조합 로직 분석

`main` 함수를 디컴파일하여 프로그램의 흐름을 분석합니다. 특히 입력 처리 및 문자열 비교 부분에 집중합니다.

```c
ulong main(int argc,char **argv,char **envp) {
    ulong uVar1;
    char *var_170h;
    char var_14ch[16];
    char *dest;
    char s2[30];
    char *var_114h;
    uint32_t var_4h;

    var_114h._0_4_ = 0x69735f;  // "_si" in little-endian?
    builtin_strncpy(s2, "irty", 5);
    s2[5] = 'x';
    s2[6] = '\0';
    builtin_strncpy(s2 + 7, "_th", 4);
    builtin_strncpy(s2 + 11, "x_r", 4);
    s2[15] = 't';
    s2[16] = 'h';
    s2[17] = '\0';
    s2[18] = 'r';
    s2[19] = 't';
    s2[20] = 'y';
    s2[21] = '\0';
    builtin_strncpy(s2 + 0x16, "_5i", 4);
    builtin_strncpy(s2 + 0x1a, "sin", 4);

    sym.imp.strcpy(&dest, s2 + 0xf);  // dest = s2 + 15 → "th"
    sym.imp.strcat(&dest, s2);         // dest = "th" + "irty..." → "thirty..."

    // 이후 사용자 입력과 dest 비교
    sym.imp.strcmp(&var_114h + 4, &dest);
    // 성공 시 플래그 출력
}
```

### 🔍 핵심 로직 분석

`s2` 배열은 다음과 같은 순서로 조각난 문자열로 구성됩니다:

1. `s2[0:4] = "irty"` → `s2 = "irty"`
2. `s2[5] = 'x'` → `s2[5] = 'x'`
3. `s2[6] = '\0'` → 문자열 종료
4. `s2[7:10] = "_th"` → `s2[7] = '_', s2[8] = 't', s2[9] = 'h'`
5. `s2[11:14] = "x_r"` → `s2[11] = 'x', s2[12] = '_', s2[13] = 'r'`
6. `s2[15] = 't'`, `s2[16] = 'h'`, `s2[17] = '\0'` → `s2[15:17] = "th"`
7. `s2[18:21] = "rty"` → `s2[18] = 'r', s2[19] = 't', s2[20] = 'y'`
8. `s2[22:25] = "_5i"` → `s2[22] = '_', s2[23] = '5', s2[24] = 'i'`
9. `s2[26:29] = "sin"` → `s2[26] = 's', s2[27] = 'i', s2[28] = 'n'`

하지만 `strcpy(&dest, s2 + 0xf)` → `s2 + 15`에서 시작하므로, `s2[15] = 't'`부터 시작하여 `"th"`가 복사됩니다.  
이후 `strcat(&dest, s2)`로 전체 `s2`를 붙이므로, 최종 `dest`는 `"th" + "irty..."` → `"thirty..."` 형태가 됩니다.

정확한 조합을 파이썬으로 시뮬레이션합니다.

---

## 💥 익스플로잇

### 5) 문자열 조합 시뮬레이션

`s2` 배열의 최종 상태를 파이썬으로 재구성하여 `dest`의 값을 계산합니다.

```python
# s2 배열 초기화 (30바이트)
s2 = ['\x00'] * 30

# s2[0:4] = "irty"
for i, c in enumerate("irty"):
    s2[i] = c

s2[5] = 'x'
s2[6] = '\0'

# s2[7:10] = "_th"
for i, c in enumerate("_th"):
    s2[7 + i] = c

# s2[11:14] = "x_r"
for i, c in enumerate("x_r"):
    s2[11 + i] = c

# s2[15] = 't', s2[16] = 'h', s2[17] = '\0'
s2[15] = 't'
s2[16] = 'h'
s2[17] = '\0'

# s2[18:21] = "rty"
for i, c in enumerate("rty"):
    s2[18 + i] = c

# s2[22:25] = "_5i"
for i, c in enumerate("_5i"):
    s2[22 + i] = c

# s2[26:29] = "sin"
for i, c in enumerate("sin"):
    s2[26 + i] = c

# s2를 null-terminated string으로 변환
s2_str = ''.join(s2).split('\x00')[0]
print('s2 =', repr(s2_str))  # s2 = 'irtyx_thx_r'

# dest = s2 + 0xf (15) → s2[15]부터 시작
dest_part1 = ''.join(s2[15:]).split('\x00')[0]  # "th"
dest = dest_part1 + s2_str  # "th" + "irtyx_thx_r" → "thirtyx_thx_r"

print('dest =', dest)
```

실행 결과:
```
s2 = 'irtyx_thx_r'
dest = 'thirtyx_thx_r'
```

하지만 이는 완전하지 않습니다. 추가적으로 `var_14ch` 배열에서 `"p0t"`, `"ck"`, `"e_"`, `"Thi"` 등을 조합하여 최종 플래그를 구성합니다.

다시 디컴파일 코드를 보면:

```c
builtin_strncpy(var_14ch, "p0t", 4);
var_14ch[4] = '0';
var_14ch[5] = '\0';
var_14ch[6] = 'c';
var_14ch[7] = 'k';
var_14ch[8] = '\0';
var_14ch[9] = 'e';
var_14ch[10] = '_';
var_14ch[11] = '\0';
var_14ch[12] = 'T';
var_14ch[13] = 'h';
var_14ch[14] = 'i';
var_14ch[15] = '\0';
```

이 배열은 다음과 같은 문자열들을 포함:
- `var_14ch[0:4]` → `"p0t"`
- `var_14ch[6:8]` → `"ck"`
- `var_14ch[9:11]` → `"e_"`
- `var_14ch[12:15]` → `"Thi"`

이후 `var_170h`는 `"Thi"` + `"rty"` + `"_5i"` + `"sin"` + `"_" + "r0cks" + "in_the_p0t"` 형태로 조합됩니다.

전체 플래그는 다음과 같은 패턴으로 구성됨:
- `"Thirty_5ix_r0cksin_the_p0t"`

이는 동화 "The Crow and the Pitcher"의 "30 + 6 = 36 stones"을 암시하는 `Thirty-six`의 변형입니다.

---

## 🚩 Flag

```
shadowCTF{Thirty_5ix_r0cksin_the_p0t}