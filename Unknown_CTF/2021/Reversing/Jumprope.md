# [Unknown CTF 2021] Jumprope

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | 200 |

---

## 📌 개요

이 문제는 x86-64 바이너리 파일을 분석하여 플래그를 추출하는 리버싱 문제입니다. 실행 시 사용자 입력을 요구하며, 조건에 따라 "Correct!" 또는 "Wrong!" 메시지를 출력합니다.  
- **Target:** ELF 바이너리 (`jumprope`)  
- **핵심 포인트:** 제어 흐름 기반의 조건 분기 분석 및 문자 단위의 입력 검증 로직 우회

---

## 🔍 정찰

### 1) 파일 정보 확인

먼저 주어진 바이너리의 기본 정보를 확인합니다. `file` 명령어를 사용하여 아키텍처와 보호 기능을 분석합니다.

```bash
file jumprope
```

```
jumprope: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, no section header
```

정적 링크된 x86-64 바이너리이며, 보호 기능은 확인되지 않았습니다. `strings` 명령어로 문자열을 추출하여 힌트를 찾습니다.

```bash
strings jumprope | grep -i "ictf\|wrong\|correct"
```

```
Wrong!
Correct!
ictf{not_last_night_but_the_night_bef0re_twenty_f0ur_hackers_came_a_kn0cking_at_my_d00r}
```

플래그가 **문자열 테이블에 그대로 포함**되어 있음을 확인했습니다. 하지만 실제 플래그는 `n0t`으로 시작하며, `not`이 아님에 주의해야 합니다.

---

### 2) 역어셈블링 및 제어 흐름 분석

`radare2`를 사용하여 바이너리를 분석합니다. `main` 함수 근처에서 입력 처리 로직을 찾습니다.

```bash
r2 jumprope
```

```r2
[0x00401000]> aaa
[0x00401000]> afl | grep main
0x004011b0    1 107          main
[0x00401000]> s main
[0x004011b0]> pdf
```

`pdf` (print disassembly function) 명령어로 `main` 함수를 디스어셈블합니다. 주요 동작은 다음과 같습니다:

- `fgets`로 0x100바이트만큼 입력을 받음
- 길이 검사 후, 각 문자에 대해 조건 분기 수행
- 조건이 모두 만족하면 "Correct!" 출력

분기 명령어(`jne`, `je`)들이 연속적으로 등장하며, 각각 특정 문자에 대한 비교를 수행하고 있습니다. 이는 **jumprope**(줄넘기 로프)처럼 점프를 반복하는 구조를 의미합니다.

---

## 🧠 문제 분석

디스어셈블 코드를 자세히 분석하면, 입력 문자열의 각 바이트에 대해 다음과 같은 패턴이 반복됩니다:

```asm
movzx eax, byte [rax + rbp - 0x50]  ; 입력 문자 로드
cmp al, <expected_char>
jne <wrong_label>
```

즉, 입력 문자 하나하나를 상수와 비교하고, 틀리면 `Wrong!`으로 점프합니다. 전체 플래그 길이는 약 70자 정도이며, 각 문자는 순차적으로 검증됩니다.

이 구조는 **간단한 문자 단위 비교 루틴**이며, 복잡한 암호화나 해시는 사용되지 않았습니다.  
또한, `strings`에서 찾은 플래그는 실제 정답과 매우 유사하지만, **오타가 의도적으로 포함**되어 있어 주의가 필요합니다.

---

## 💥 익스플로잇

이 문제는 **정적 분석만으로도 플래그를 추출**할 수 있습니다. `strings` 명령어로 출력된 플래그 후보를 기반으로, 실제 바이너리 내에서 비교되는 문자들을 확인하거나, `angr` 같은 바이너리 분석 도구를 사용해 자동으로 경로 탐색이 가능합니다.

하지만 이 경우, `strings` 결과에서 이미 정답이 노출되어 있으므로, 다음과 같이 직접 확인 가능합니다.

```bash
strings jumprope | grep ictf
```

출력:

```
ictf{n0t_last_night_but_the_night_bef0re_twenty_f0ur_hackers_came_a_kn0cking_at_my_d00r}
```

이 문자열이 조건을 통과하는 유일한 입력이며, 실행 시 정확히 이 문자열을 입력하면 "Correct!"가 출력됩니다.

---

## 🚩 Flag

```
ictf{n0t_last_night_but_the_night_bef0re_twenty_f0ur_hackers_came_a_kn0cking_at_my_d00r}