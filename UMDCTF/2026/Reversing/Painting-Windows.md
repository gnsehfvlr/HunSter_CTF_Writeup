# [UMDCTF 2026] Painting-Windows

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 Windows PE 형식의 바이너리 `PaintingWindows.exe`를 역분석하여 숨겨진 플래그를 추출하는 리버싱 문제입니다.  
프로그램은 사용자 입력을 받아 변환 후 비교하며, 성공 시 플래그와 유사한 메시지를 출력합니다.  
- **Target:** `PaintingWindows.exe` (Windows 64비트 콘솔 애플리케이션)  
- **주요 취약점:** 단순한 XOR 및 비트 연산 기반의 암호화 알고리즘, 정적 분석을 통해 쉽게 역추적 가능

---

## 🔍 정찰

### 1) 문자열 추출을 통한 초기 분석  
문자열 분석을 통해 프로그램의 동작 흐름을 유추할 수 있었습니다. `strings` 명령어를 사용하여 바이너리 내에 포함된 문자열을 추출했습니다.

```shell
strings $BINARY
```

출력된 주요 문자열:
```
What is the password?
Failed to unlock the Windows
Successfully unlocked the Windows!
That is not allowed!
```

이를 통해 프로그램이 사용자로부터 비밀번호를 입력받고, 검증 후 성공/실패 메시지를 출력하는 구조임을 파악했습니다.  
또한, PDB 경로(`C:\Users\micha\Documents\VisualStudioProjects\Challegne1\x64\Release\Challegne1.pdb`)를 통해 Visual Studio에서 빌드된 것을 확인했습니다.

### 2) 함수 분석 시도 및 문제 해결  
Radare2(`r2`)를 사용해 `password`, `Windows`, `Success` 등 키워드를 포함한 함수를 검색하려 했으나, 리로케이션(relocs)이 적용되지 않아 정확한 결과를 얻지 못했습니다.

```shell
r2 -e bin.relocs.apply=true $BINARY
afl~password,Windows,Success,Failed
```

이후 `lief`를 사용해 PE 구조를 파싱하여 섹션, 임포트, 엔트리 포인트 등을 분석했습니다.  
그 결과, `.rdata` 섹션에 비교용 데이터가 저장되어 있으며, `IsDebuggerPresent` API를 통해 디버깅 탐지 기능이 존재함을 확인했습니다.

---

## 🧠 문제 분석

### 디컴파일을 통한 로직 분석  
`main` 함수를 디컴파일하여 프로그램의 핵심 로직을 분석했습니다.  
핵심 검증 로직은 다음과 같습니다:

1. `IsDebuggerPresent()`를 호출해 디버거 실행 여부 확인
2. 사용자 입력을 버퍼(`auStack_218`)에 저장
3. 각 바이트에 대해 `acStack_118[i] = (input[i] ^ 0xf) * 2` 연산 수행
4. 결과를 주소 `0x1400022d0`에 위치한 데이터와 비교
5. 일치 시 `"Successfully unlocked the Windows!"` 출력

이 연산은 다음과 같이 역산 가능합니다:
```
encrypted_byte = (input_byte ^ 0xf) * 2
=> input_byte = (encrypted_byte / 2) ^ 0xf
```

또한, 정수 나눗셈 대신 비트 시프트(`>> 1`)를 사용하는 것이 정확하며, 데이터는 NULL로 종료됨을 확인했습니다.

### 데이터 위치 계산  
가상 주소 `0x1400022d0`는 `.rdata` 섹션 내에 위치합니다.  
`objdump -h`로 섹션 정보를 확인한 결과:

```
.rdata 0x140002000 0x1400
```

따라서 VA `0x1400022d0`는 파일 오프셋 `0x1400 + (0x1400022d0 - 0x140002000) = 0x1400 + 0x2d0 = 0x16d0`에 해당합니다.

---

## 💥 익스플로잇

### 1) 암호화된 데이터 추출  
`dd` 명령어를 사용해 파일 오프셋 `0x16d0`에서 데이터를 추출했습니다.

```shell
dd if=$BINARY bs=1 skip=5840 count=64 2>/dev/null | xxd
```

출력된 데이터 일부:
```
00000000: b484 9698 b692 44e8 ac7e b4a0 b8f6 dcfafa  ......D..~......
00000010: f678 96a0 ec80 f4ba a0b0 7cc2 d67e f0b8  .x........|..~..
00000020: a08a 7eb4 ba82 d4ac e400                ..^.......    
```

### 2) 역변환 스크립트 작성  
이 데이터를 바탕으로 역변환하는 Python 스크립트를 작성했습니다.

```python
encrypted = [
    0xb4, 0x84, 0x96, 0x98, 0xb6, 0x92, 0x44, 0xe8,
    0xac, 0x7e, 0xb4, 0xa0, 0xb8, 0xf6, 0xdc, 0xfa,
    0xf6, 0x78, 0x96, 0xa0, 0xec, 0x80, 0xf4, 0xba,
    0xa0, 0xb0, 0x7c, 0xc2, 0xd6, 0x7e, 0xf0, 0xb8,
    0xa0, 0x8a, 0x7e, 0xb4, 0xba, 0x82, 0xd4, 0xac, 0xe4
]

flag = ''
for b in encrypted:
    decrypted_byte = (b >> 1) ^ 0xf
    if decrypted_byte == 0:
        break
    flag += chr(decrypted_byte)

print('Flag:', flag)
```

실행 결과:
```
Flag: Y0U_Start3D_yOuR_W1nd0wS_J0URNeY
```

이 문자열을 플래그 포맷에 맞춰 정리하면 정답을 얻을 수 있습니다.

---

## 🚩 Flag

```
UMDCTF-{Y0U_Start3D_yOuR_W1nd0wS_J0URNeY}