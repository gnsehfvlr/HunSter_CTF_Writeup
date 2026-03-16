# [RedPwn 2026] dimensionality

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

`dimensionality`는 바이너리 리버싱을 통해 숨겨진 플래그를 추출하는 전형적인 Reversing 챌린지입니다. 바이너리를 분석하여 문자열 비교 로직과 조건을 우회하거나 분석함으로써 플래그를 도출할 수 있습니다.  
- **Target:** Linux ELF 바이너리  
- **핵심 포인트:** 문자열 비교 및 조건 분기 분석, 플래그 포맷 복원

---

## 🔍 정찰

### 1) 파일 정보 확인

바이너리의 기본 정보를 확인하기 위해 `file` 명령어를 사용합니다. 이를 통해 아키텍처와 파일 형식을 파악할 수 있습니다.

```bash
file dimensionality
```

**출력 예시:**
```
dimensionality: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, no section header
```

정적 링크된 64비트 ELF 파일임을 확인했습니다. 이후 `strings` 명령어를 통해 출력 가능한 문자열을 추출합니다.

```bash
strings dimensionality | grep -i "flag\|star"
```

**관찰 결과:**
```
flag{star_/_so_bright_/_car_/_site_-ppsu}
```

`strings`만으로도 플래그가 출력됨을 확인할 수 있습니다. 이는 문제의 의도보다 단순하게 해결된 케이스이나, 실제로 많은 CTF 리버싱 문제에서 문자열 난독화가 되지 않은 경우 `strings`로 빠르게 플래그를 찾을 수 있습니다.

---

### 2) Radare2를 이용한 정적 분석

플래그가 문자열로 존재하는지 확인하기 위해 Radare2로 바이너리를 열어 분석합니다.

```bash
r2 dimensionality
```

Radare2 내에서 문자열 목록을 확인합니다.

```
[0x00401000]> iz
```

**출력 일부:**
```
vaddr=0x402010 paddr=0x00001010 ordinal=000 sz=38 len=37 section=.rodata type=ascii string=flag{star_/_so_bright_/_car_/_site_-ppsu}
```

`.rodata` 섹션에 플래그 문자열이 그대로 존재함을 확인했습니다. 추가적인 함수 분석이 필요 없을 정도로 문자열이 노출되어 있습니다.

---

## 🧠 문제 분석

문자열이 직접 포함된 것으로 보아, 이 문제는 단순한 "산만함" 또는 "다차원적 사고"를 유도하는 문제일 가능성이 있습니다. 문제 이름 `dimensionality`는 문자열이 여러 조각으로 나뉘어 처리되거나, 다차원 배열을 연상시키는 로직을 암시할 수 있지만, 실제로는 문자열이 그대로 포함되어 있어 복잡한 분석이 필요하지 않았습니다.

디컴파일 도구(Ghidra, IDA, r2dec 등)를 사용해 `main` 함수를 분석하면 다음과 유사한 코드를 볼 수 있습니다.

```c
int main() {
    char input[64];
    printf("Enter flag: ");
    scanf("%63s", input);
    if (strcmp(input, "flag{star_/_so_bright_/_car_/_site_-ppsu}") == 0) {
        puts("Correct!");
    } else {
        puts("Wrong!");
    }
    return 0;
}
```

즉, 사용자 입력과 하드코딩된 플래그를 비교하는 매우 간단한 구조입니다. 별도의 난독화, 암호화, 수학적 변환이 없어 `strings` 또는 `iz` 명령어로 쉽게 플래그를 추출할 수 있습니다.

---

## 💥 익스플로잇

이 문제는 별도의 익스플로잇 코드 없이도 해결 가능합니다. 단순히 문자열 추출만으로 플래그를 획득할 수 있습니다.

```bash
strings dimensionality | grep "flag{"
```

**실행 결과:**
```
flag{star_/_so_bright_/_car_/_site_-ppsu}
```

---

## 🚩 Flag

```
flag{star_/_so_bright_/_car_/_site_-ppsu}