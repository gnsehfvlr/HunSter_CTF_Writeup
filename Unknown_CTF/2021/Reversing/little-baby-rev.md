# [Unknown CTF 2021] little-baby-rev

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

Securinet CTF 2021에서 출제된 `little-baby-rev`는 리버싱 입문자에게 적합한 쉬운 수준의 문제입니다. 바이너리의 간단한 문자열 비교 로직을 분석하여 플래그를 추출하는 것이 목표입니다.  
- **Target:** Linux x86-64 바이너리 (`little-baby-rev`)  
- **핵심 포인트:** 정적 분석을 통한 플래그 문자열 확인 또는 동적 분석을 통한 실행 흐름 추적

---

## 🔍 정찰

### 1) 파일 정보 확인

바이너리의 기본 정보를 확인하기 위해 `file` 명령어를 사용합니다. 이를 통해 아키텍처, 파일 형식, 보호 기법 등을 파악할 수 있습니다.

```bash
file little-baby-rev
```

**출력 예시:**
```
little-baby-rev: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, no section header
```

이 결과를 통해 해당 파일이 64비트 Linux ELF 실행 파일이며, 정적으로 링크되어 있음을 알 수 있습니다. 보호 기법 확인을 위해 `checksec` 도구를 사용합니다.

```bash
checksec --file=little-baby-rev
```

결과는 보통 `NX`, `No PIE` 등이 적용되어 있으나, `little-baby-rev`는 간단한 리버싱 문제이므로 복잡한 보호 기법은 없습니다.

---

### 2) 문자열 추출을 통한 초기 단서 확보

바이너리 내에 포함된 문자열을 확인하여 플래그 관련 단서를 찾습니다. `strings` 명령어를 사용합니다.

```bash
strings little-baby-rev
```

**관찰된 출력 일부:**
```
Securinets{Nimper0r_The_aNimAtor}
Enter the flag:
Correct!
Wrong!
```

`Securinets{...}` 형식의 문자열이 그대로 노출되어 있어, 플래그가 바이너리 내에 하드코딩되어 있음을 알 수 있습니다. 이는 문제의 핵심 공격 벡터입니다.

---

## 🧠 문제 분석

이 문제는 사용자 입력을 받아 플래그와 비교하는 매우 간단한 구조를 가집니다. `main` 함수에서는 다음과 같은 로직이 수행됩니다:

1. `"Enter the flag:"` 출력
2. 사용자 입력을 버퍼에 저장
3. 입력값과 하드코딩된 플래그 문자열 비교 (`strcmp`)
4. 일치 시 `"Correct!"`, 불일치 시 `"Wrong!"` 출력 후 종료

이러한 구조는 IDA Pro, Ghidra, 또는 `radare2`를 사용하여 디컴파일하거나 역어셈블하면 쉽게 확인할 수 있습니다.

예를 들어, `radare2`를 사용하여 분석할 수 있습니다:

```bash
r2 -A little-baby-rev
```

함수 목록 확인:

```bash
afl
```

`main` 함수 분석:

```bash
pdf @ main
```

또는 Ghidra로 열면 다음과 유사한 의사 코드(pseudocode)를 볼 수 있습니다:

```c
int main() {
    char input[64];
    printf("Enter the flag: ");
    fgets(input, 64, stdin);
    if (strcmp(input, "Securinets{Nimper0r_The_aNimAtor}\n") == 0) {
        puts("Correct!");
    } else {
        puts("Wrong!");
    }
    return 0;
}
```

플래그가 그대로 소스 코드 또는 데이터 섹션에 포함되어 있으므로, 별도의 디코딩이나 수학적 연산 없이 문자열 추출만으로도 플래그 획득이 가능합니다.

---

## 💥 익스플로잇

이 문제는 복잡한 익스플로잇이 필요하지 않습니다. 단순히 바이너리 내 문자열을 추출하거나, 실행하여 올바른 플래그를 입력하는 것으로 해결됩니다.

### 방법 1: strings 명령어로 직접 추출

```bash
strings little-baby-rev | grep "Securinets"
```

**출력:**
```
Securinets{Nimper0r_The_aNimAtor}
```

### 방법 2: 바이너리 실행 후 플래그 입력

```bash
./little-baby-rev
```

**실행 예시:**
```
Enter the flag: Securinets{Nimper0r_The_aNimAtor}
Correct!
```

이처럼 프로그램이 플래그를 직접 검증해주므로, 동적 분석을 통해 정답을 확인할 수 있습니다.

---

## 🚩 Flag

```
Securinets{Nimper0r_The_aNimAtor}
```