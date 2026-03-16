# [Angstrom 2026] Jailbreak

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

주어진 바이너리 `jailbreak`를 역분석하여 플래그를 추출하는 문제입니다. 프로그램은 사용자 입력을 받아 특정 조건을 검사하며, 조건을 만족해야 플래그를 출력합니다.  
- **Target:** `jailbreak` ELF 바이너리  
- **핵심 포인트:** 제어 흐름 분석, 문자열 비교 로직 우회, 의도치 않은 솔루션(unintended solution) 활용

---

## 🔍 정찰

### 1) 파일 정보 확인
바이너리의 기본 정보를 확인하기 위해 `file` 명령어를 사용합니다.

```bash
file jailbreak
```

**출력:**
```
jailbreak: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, no section header
```

정적으로 링크된 64비트 ELF 파일이며, 보호 기능이 제한적으로 적용되어 있을 가능성이 큽니다.

### 2) 문자열 분석
바이너리 내에 포함된 문자열을 확인하여 힌트를 찾습니다.

```bash
strings jailbreak
```

**관찰된 출력 일부:**
```
Enter the flag: 
Correct!
Wrong!
actf{guess_kmh_still_has_unintended_solutions}
```

플래그가 **바이너리 내부에 평문으로 존재**한다는 것을 확인할 수 있습니다. 이는 역분석 없이도 `strings` 명령어로 직접 플래그를 추출할 수 있음을 의미합니다.

---

## 🧠 문제 분석

이 문제는 reversing 카테고리지만, 플래그가 바이너리에 **직접 포함되어 있어** 복잡한 분석이 필요하지 않습니다.  
일반적인 의도된 솔루션은 입력값을 받아 해시 또는 비교 연산을 수행하고, 조건을 만족하면 "Correct!"를 출력하도록 설계되어 있을 것으로 예상됩니다.

그러나 `strings` 결과에서 플래그가 그대로 노출되어 있어, **의도치 않은(unintended) 솔루션**으로 플래그를 즉시 획득할 수 있습니다.

이러한 현상은 CTF 문제에서 종종 발생하며, 개발자가 디버깅 문자열이나 테스트 플래그를 제거하지 않았을 때 나타납니다.

---

## 💥 익스플로잇

문자열 추출 명령어를 사용하여 플래그를 즉시 획득합니다.

```bash
strings jailbreak | grep "actf{"
```

**실행 결과:**
```
actf{guess_kmh_still_has_unintended_solutions}
```

이 명령어는 바이너리 내 모든 문자열을 출력한 후, `actf{`로 시작하는 문자열만 필터링하여 플래그를 노출시킵니다.

---

## 🚩 Flag

```
actf{guess_kmh_still_has_unintended_solutions}
```