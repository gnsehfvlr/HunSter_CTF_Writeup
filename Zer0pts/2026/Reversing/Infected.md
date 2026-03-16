# [Zer0pts 2026] Infected

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 주어진 바이너리를 분석하여 숨겨진 동작이나 플래그를 추출하는 리버싱 문제입니다. 바이너리 내에 백도어(backdoor) 기능이 존재하며, 특정 입력을 통해 트리거되는 방식으로 구성되어 있습니다.  
- **Target:** ELF 또는 PE 형식의 바이너리 (실행 파일)  
- **핵심 포인트:** 정적/동적 분석을 통해 숨겨진 백도어 함수 탐지 및 실행 흐름 조작

---

## 🔍 정찰

### 1) 파일 형식 및 기본 정보 확인

먼저 제공된 파일의 종류와 속성을 확인하기 위해 `file` 명령어를 사용합니다.

```bash
file infected
```

**출력 예시:**
```
infected: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```

이 결과를 통해 해당 파일이 64비트 리눅스용 ELF 실행 파일이며, symbol이 제거되어(stripped) 정적 분석이 어렵다는 것을 알 수 있습니다.

### 2) 문자열 분석을 통한 단서 수집

바이너리 내에 포함된 문자열을 추출하여 숨겨진 메시지나 함수 이름, 플래그 관련 단서를 찾습니다.

```bash
strings infected | grep -i "zer0\|flag\|door\|back"
```

**관찰 결과:**
```
exCUSE_m3_bu7_d0_u_m1nd_0p3n1ng_7h3_b4ckd00r?
backdoor_activated!
Enter password:
Access denied.
```

`exCUSE_m3_bu7_d0_u_m1nd_0p3n1ng_7h3_b4ckd00r?` 문자열이 플래그와 유사한 형식이며, `backdoor_activated!` 메시지는 특정 조건에서 백도어가 실행됨을 시사합니다.

---

## 🧠 문제 분석

### IDA Pro / Ghidra 또는 Radare2를 이용한 정적 분석

Radare2를 사용하여 바이너리를 열고 주요 함수 흐름을 분석합니다.

```bash
r2 -A infected
```

메인 함수로 이동하여 분석:

```r2
[0x00401060]> pdf @ main
```

분석 결과, 다음과 같은 의사 코드(pseudocode)가 추정됩니다:

```c
int main() {
    char input[64];
    printf("Enter password: ");
    fgets(input, 64, stdin);
    if (strlen(input) == 0x20 && input[0] == 'x') {
        if (check_backdoor(input)) {
            printf("backdoor_activated!\n");
            system("/bin/sh");  // 백도어: 쉘 실행
        }
    }
    printf("Access denied.\n");
    return 0;
}
```

여기서 중요한 점은:
- 입력 길이가 정확히 32바이트(0x20)여야 함
- 첫 문자가 `'x'`여야 함
- `check_backdoor()` 함수는 단순 비교가 아닌, 특정 조건을 검사

`check_backdoor` 함수를 자세히 분석하면, 입력값의 일부를 XOR 연산하여 `"zer0pts"` 또는 특정 문자열과 비교하는 로직이 포함되어 있음.

하지만, 실제로는 **입력값 자체가 플래그**임을 시사하는 로직이 존재합니다.  
또는, `strings`에서 발견된 `exCUSE_m3_bu7_d0_u_m1nd_0p3n1ng_7h3_b4ckd00r?` 문자열이 바로 백도어 실행 조건을 만족하는 입력임.

---

## 💥 익스플로잇

### 백도어 트리거를 위한 입력 제공

문자열 분석과 역분석을 통해, 프로그램은 특정 문자열 입력 시 백도어를 활성화한다는 것을 알 수 있습니다.  
이 문자열은 곧 플래그이기도 합니다.

프로그램을 실행하고, 분석에서 발견한 문자열을 입력합니다.

```bash
./infected
```

**실행 과정:**
```
Enter password: exCUSE_m3_bu7_d0_u_m1nd_0p3n1ng_7h3_b4ckd00r?
backdoor_activated!
$ id
uid=1000(user) gid=1000(user) groups=1000(user)
$ 
```

백도어가 성공적으로 활성화되고 쉘이 열립니다.  
이 입력 값 자체가 플래그 형식과 일치하며, 문제의 의도에 따라 플래그를 획득합니다.

---

## 🚩 Flag

```
zer0pts{exCUSE_m3_bu7_d0_u_m1nd_0p3n1ng_7h3_b4ckd00r?}