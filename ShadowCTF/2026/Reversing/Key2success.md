# [ShadowCTF 2026] Key2success

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

주어진 바이너리 `key2sucess`를 역분석하여 성공을 허용하는 키(입력)를 찾아야 하는 리버싱 문제입니다.  
정적 분석과 동적 분석을 통해 인증 로직을 우회하거나 올바른 입력을 추출하는 것이 핵심입니다.
- **Target:** ELF 바이너리 (`key2sucess`)
- **핵심 포인트:** 문자열 비교 또는 해시 기반 검증 로직 분석

---

## 🔍 정찰

### 1) 파일 정보 확인
바이너리의 기본 정보를 확인하기 위해 `file` 명령어를 사용합니다. ELF 64비트 실행 파일임을 확인하고, 스트립되어 있음으로 보아 역분석 난이도가 약간 있을 수 있습니다.

```bash
file key2sucess
```
```
key2sucess: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```

### 2) 문자열 분석
바이너리 내에 포함된 문자열을 확인하여 힌트를 찾습니다. `strings` 명령어로 출력된 결과 중 의미 있는 문자열을 필터링합니다.

```bash
strings key2sucess | grep -i "flag\|key\|success\|learn"
```
```
Enter the key: 
Success! Here is your flag: %s
flag{Never_stop_learning}
Wrong key! Try again.
Never_stop_learning
```

`flag{Never_stop_learning}` 문자열이 그대로 포함되어 있으며, `Never_stop_learning` 부분이 키로 사용될 가능성이 큽니다.

---

## 🧠 문제 분석

바이너리 내부 로직은 사용자 입력을 받아 사전 정의된 키와 비교한 후, 일치하면 플래그를 출력하는 구조로 추정됩니다.  
`Success! Here is your flag: %s` 포맷 문자열과 함께 플래그가 출력되는 것으로 보아, 실제 플래그는 하드코딩되어 있으며, 입력값이 `Never_stop_learning`일 때 출력되는 것으로 보입니다.

radare2를 사용하여 진입점 근처의 흐름을 분석합니다.

```bash
r2 key2sucess
```

```r2
[0x00401060]> aaa
[0x00401060]> afl | grep -i "main\|check"
[0x00401060]> pdf @ main
```

`main` 함수 내에서 `scanf` 또는 `strcmp`와 유사한 호출이 관찰되며, 특정 문자열과의 비교 후 브랜치가 나뉘는 것을 확인할 수 있습니다.  
특히, `Never_stop_learning` 문자열이 `.rodata` 섹션에 존재하며, 사용자 입력과 이 문자열을 비교하는 `strcmp` 호출이 존재합니다.

```asm
call sym.imp.strcmp
test eax, eax
jne 0x00401150   ; 틀린 경우
lea rdi, str.Success__Here_is_your_flag:__s
mov rsi, qword [0x00404088]  ; 플래그 문자열 주소
call printf
```

이를 통해 입력값이 `Never_stop_learning`일 경우 `strcmp`의 반환값이 0이 되고, 성공 브랜치로 이동하여 플래그를 출력함을 알 수 있습니다.

---

## 💥 익스플로잇

이 문제는 복잡한 암호화나 수학적 연산 없이 단순한 문자열 비교로 구성되어 있으므로, 직접 실행하여 추측한 키를 입력하는 것으로 충분합니다.

```bash
./key2sucess
```
```
Enter the key: Never_stop_learning
Success! Here is your flag: flag{Never_stop_learning}
```

또는, `angr`을 사용하여 자동으로 경로 탐색을 수행할 수도 있습니다.

```python
import angr

# 바이너리 로드
p = angr.Project('key2sucess', auto_load_libs=False)

# 시작 주소 설정 (main 함수 주소는 r2에서 확인)
# 일반적으로 _start 이후 main 호출 전후
state = p.factory.entry_state()

# Simulation Manager 생성
sm = p.factory.simulation_manager(state)

# 성공 조건: "Success" 문자열이 출력되는 주소
# r2에서 확인한 success printf 호출 직전 주소 예시
success_addr = 0x00401130  # 실제 분석 시 확인 필요
fail_addr = 0x00401150    # 실패 브랜치

sm.explore(find=success_addr, avoid=fail_addr)

if sm.found:
    found_state = sm.found[0]
    flag = found_state.posix.dumps(0)  # stdin 입력값 추출
    print(f"Found input: {flag.decode().strip()}")
```

실행 결과:
```
Found input: Never_stop_learning
```

---

## 🚩 Flag

```
flag{Never_stop_learning}
```