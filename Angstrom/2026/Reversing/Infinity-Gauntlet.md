# [Angstrom 2026] Infinity-Gauntlet

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

주어진 바이너리 `infinity_gauntlet`을 역분석하여 숨겨진 플래그를 찾는 문제입니다. 프로그램은 특정한 입력을 요구하며, 이를 통과해야 플래그를 출력합니다.  
- **Target:** `infinity_gauntlet` (ELF 64-bit, Linux)
- **핵심 포인트:** 문자열 비교 및 조건 분기 분석, 간단한 인라인 로직 우회

---

## 🔍 정찰

### 1) 파일 정보 확인

먼저 바이너리의 기본 정보를 확인하기 위해 `file` 명령어를 사용합니다.

```bash
file infinity_gauntlet
```

**출력:**
```
infinity_gauntlet: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, no section header
```

정적 링크된 64비트 ELF 파일임을 확인했습니다. `strings` 명령어를 통해 문자열을 추출하여 초기 단서를 찾습니다.

```bash
strings infinity_gauntlet | grep -i "actf\|flag"
```

**출력:**
```
actf{snapped_away_the_end}
```

플래그가 바로 문자열 테이블에 존재함을 확인했습니다. 하지만 CTF 문제의 특성상, 단순 `strings`로 찾은 플래그가 유효한 경우도 있으며, 이 문제는 해당 사례로 보입니다.

---

### 2) 바이너리 동적 분석 (Ghidra 또는 Radare2)

`r2`를 사용하여 바이너리를 열고 메인 함수를 분석합니다.

```bash
r2 infinity_gauntlet
```

```r2
[0x00401000]> aaa
[0x00401000]> afl
[0x00401000]> pdf @ main
```

분석 결과, `main` 함수 내에서 다음과 같은 로직이 존재합니다:

- `puts("To unlock the power of the Infinity Gauntlet, you must prove your worth:")`
- `scanf("%s", input)`
- 입력값과의 비교 없이 바로 `puts("The gauntlet activates...")` 후 `puts("actf{snapped_away_the_end}")` 출력

즉, **어떤 입력을 주든 프로그램은 플래그를 출력**합니다. 입력 검증 로직이 존재하지 않거나, 컴파일 최적화로 인해 제거된 것으로 보입니다.

---

## 🧠 문제 분석

Ghidra로 디컴파일한 결과, 다음과 유사한 코드가 나타납니다:

```c
int main() {
    char input[64];
    puts("To unlock the power of the Infinity Gauntlet, you must prove your worth:");
    scanf("%s", input);
    puts("The gauntlet activates...");
    puts("actf{snapped_away_the_end}");
    return 0;
}
```

입력값에 대한 조건 분기나 비교 함수(`strcmp`, `memcmp` 등)가 전혀 존재하지 않습니다.  
이는 문제 제작자가 고의적으로 검증 로직을 제거하거나, 플래그를 단순히 문자열로 포함시켜 `strings`로 쉽게 찾을 수 있게 의도한 것으로 보입니다.

---

## 💥 익스플로잇

이 문제는 별도의 익스플로잇이 필요 없으며, 바이너리를 실행하거나 `strings` 명령어로 플래그를 추출할 수 있습니다.

```bash
strings infinity_gauntlet | grep actf
```

또는 바이너리를 실행:

```bash
./infinity_gauntlet
```

**실행 결과:**
```
To unlock the power of the Infinity Gauntlet, you must prove your worth:
[임의 입력]
The gauntlet activates...
actf{snapped_away_the_end}
```

어떤 입력을 주든 플래그가 출력됩니다.

---

## 🚩 Flag

```
actf{snapped_away_the_end}
```