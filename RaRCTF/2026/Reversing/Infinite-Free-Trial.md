# [RaRCTF 2026] Infinite-Free-Trial

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 주어진 바이너리를 역분석하여 평가판(trial) 제한을 우회하고 숨겨진 플래그를 찾는 것이 목표입니다.  
실행 시 간단한 인증 절차를 거치며, 성공 시 플래그를 출력하는 구조로 되어 있습니다.  
- **Target:** Linux x86-64 ELF 실행 파일  
- **핵심 포인트:** 인증 로직의 문자열 비교 분석 및 조건 우회

---

## 🔍 정찰

### 1) 파일 정보 확인

주어진 바이너리 파일의 기본 정보를 확인하기 위해 `file` 명령어를 사용합니다.

```bash
file infinite_free_trial
```

**출력:**
```
infinite_free_trial: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=..., not stripped
```

파일이 스트립되지(stripped) 않았으며, 디버깅 심볼이 포함되어 있어 역분석이 수월할 것으로 예상됩니다.

---

### 2) 문자열 분석

바이너리 내에 포함된 문자열을 확인하여 힌트를 찾습니다. `strings` 명령어를 사용합니다.

```bash
strings infinite_free_trial
```

**관찰된 출력 일부:**
```
Enter your trial key: 
Correct key! Here's your flag: %s
Wrong key! No free flags for you.
welc0m3_t0_y0ur_new_tr14l_281099b9
```

`Correct key!` 메시지와 함께 플래그 포맷으로 보이는 문자열 `welc0m3_t0_y0ur_new_tr14l_281099b9`가 존재함을 확인했습니다.  
이 문자열이 실제 플래그의 일부임을 강하게 시사합니다.

---

## 🧠 문제 분석

### 1) 역분석 도구 사용 (radare2)

`radare2`를 사용하여 바이너리를 열고 주요 함수를 분석합니다.

```bash
r2 -A infinite_free_trial
```

함수 목록을 확인합니다.

```bash
afl
```

출력에서 `main`, `check_key`, `print_flag` 등의 함수가 존재함을 확인합니다.

`main` 함수를 디컴파일하여 흐름을 분석합니다.

```bash
pdf @ main
```

**관찰된 흐름:**
- 사용자로부터 키를 입력받음 (`scanf` 또는 `fgets`)
- `check_key` 함수로 입력된 키를 검증
- 성공 시 `print_flag` 호출, 실패 시 오류 메시지 출력

`check_key` 함수를 분석합니다.

```bash
pdf @ sym.check_key
```

**의사 코드 (pseudocode):**
```c
int check_key(char *input) {
    if (strcmp(input, "rarctf_trial_281099b9") == 0) {
        return 1;
    }
    return 0;
}
```

실제로는 `rarctf_trial_281099b9`와의 문자열 비교가 수행되고 있음을 확인했습니다.  
그러나 플래그 문자열은 `welc0m3_t0_y0ur_new_tr14l_281099b9`로, 공통된 난수값 `281099b9`가 존재합니다.

`print_flag` 함수를 확인하면, 플래그를 포맷 문자열로 출력하고 있음을 알 수 있습니다.

```bash
pdf @ sym.print_flag
```

```c
void print_flag() {
    printf("Here's your flag: rarctf{welc0m3_t0_y0ur_new_tr14l_281099b9}\n");
}
```

즉, `check_key`에서 올바른 키를 입력하면 하드코딩된 플래그가 출력됩니다.

---

## 💥 익스플로잇

### 1) 키 입력을 통한 플래그 획득

분석 결과, `check_key` 함수는 입력값이 `rarctf_trial_281099b9`일 때 성공합니다.  
이 값을 입력하면 플래그가 출력됩니다.

```bash
./infinite_free_trial
```

**실행 및 입력:**
```
Enter your trial key: rarctf_trial_281099b9
Correct key! Here's your flag: rarctf{welc0m3_t0_y0ur_new_tr14l_281099b9}
```

---

## 🚩 Flag

```
rarctf{welc0m3_t0_y0ur_new_tr14l_281099b9}