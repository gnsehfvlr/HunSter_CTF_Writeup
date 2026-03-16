# [Angstrom 2026] Jailbreak

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

주어진 바이너리 `jailbreak`를 역분석하여 숨겨진 플래그를 추출하는 문제입니다. 프로그램은 입력을 검증하는 복잡한 로직을 포함하고 있으며, 일부 의도치 않은 우회 방법(unintended solution)을 통해 플래그를 빠르게 획득할 수 있었습니다.  
- **Target:** Linux x86-64 바이너리 (`jailbreak`)  
- **핵심 포인트:** 제어 흐름 분석, 문자열 참조 탐색, unintended solution 활용

---

## 🔍 정찰

### 1) 파일 정보 확인 및 초기 분석

바이너리의 기본 속성을 확인하기 위해 `file` 명령어를 사용했습니다. 이를 통해 아키텍처와 보호 기능을 파악할 수 있습니다.

```bash
file jailbreak
```

**출력:**
```
jailbreak: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, no section header
```

정적으로 링크된 ELF 파일이며, 보호 기능 확인을 위해 `checksec`을 실행합니다.

```bash
checksec --file=jailbreak
```

**관찰:**  
- `NX enabled`, `No PIE`, `Stack Canary 없음`  
- 역분석이 가능하나, 복잡한 제어 흐름을 가진 것으로 추정

---

### 2) 문자열 분석을 통한 단서 수집

바이너리 내에 포함된 문자열을 `strings` 명령어로 추출하여 초기 단서를 찾습니다.

```bash
strings jailbreak | grep -i "actf\|flag\|success\|wrong"
```

**관찰된 출력:**
```
actf{guess_kmh_still_has_unintended_solutions}
Wrong.
Success!
Enter input:
```

플래그가 **바이너리 내에 평문으로 존재**한다는 것을 확인했습니다. 이는 문제의 의도된 솔루션(입력 검증 로직 우회 또는 역분석)과는 별개로, 단순한 `strings` 명령어만으로도 플래그를 획득할 수 있음을 의미합니다.

---

## 🧠 문제 분석

정적으로 링크된 바이너리이므로, `radare2` 또는 `Ghidra`를 사용해 제어 흐름을 분석할 수 있습니다. `main` 함수 근처에서 사용자 입력을 받고 비교하는 로직이 존재할 것으로 예상됩니다.

`radare2`를 사용해 분석을 시작합니다.

```bash
r2 -A jailbreak
```

함수 목록을 확인합니다.

```
afl
```

`main` 함수 또는 유사한 진입점에서 `scanf` 또는 `read`와 같은 입력 함수 호출을 찾습니다. 이후, 비교 연산이나 문자열 비교(`strcmp`)가 있는지 확인합니다.

디컴파일 도구(Ghidra)로 분석 시, 아래와 유사한 로직이 발견됩니다:

```c
int main() {
    char input[64];
    printf("Enter input: ");
    scanf("%63s", input);
    if (check_input(input)) {
        printf("Success!\n");
    } else {
        printf("Wrong.\n");
    }
    return 0;
}
```

`check_input` 함수는 매우 복잡한 조건을 검사하며, 여러 개의 수학적 조건과 문자 단위 비교를 포함합니다. 이는 intended solution으로, Z3 같은 SMT solver를 사용해 조건을 풀도록 유도하는 것으로 보입니다.

그러나, **플래그 문자열이 `.rodata` 섹션에 평문으로 포함되어 있어**, 복잡한 분석 없이도 `strings`만으로 플래그를 추출할 수 있었습니다.

---

## 💥 익스플로잇

이 문제는 **unintended solution**을 통해 매우 간단히 해결됩니다. 의도된 접근은 제어 흐름 분석과 조건 해석이었지만, 플래그가 바이너리에 노출된 상태였기 때문에 간단한 문자열 추출로 해결 가능합니다.

```bash
strings jailbreak | grep "actf{"
```

**실행 결과:**
```
actf{guess_kmh_still_has_unintended_solutions}
```

이처럼, 복잡한 리버싱 없이도 플래그를 즉시 획득할 수 있었습니다.

---

## 🚩 Flag

```
actf{guess_kmh_still_has_unintended_solutions}
```