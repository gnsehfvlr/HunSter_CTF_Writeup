# [FooBar 2026] numerical-computing

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 Fortran으로 컴파일된 ELF 바이너리를 리버싱하여 숨겨진 플래그를 찾는 문제입니다. 바이너리는 사용자 입력을 받아 수치 기반의 검증 로직을 수행하며, 입력이 조건을 만족하면 성공 메시지를 출력합니다.  
- **Target:** Fortran 컴파일된 바이너리 `try`  
- **주요 취약점:** 입력 값에 대한 단순한 XOR 및 비트 연산 기반의 검증 로직으로, 정적 분석을 통해 역산 가능

---

## 🔍 정찰

### 1) 초기 정보 수집
바이너리의 기본 정보를 확인하기 위해 `file`, `strings`, `ls` 명령어를 사용하여 분석을 시작했습니다. 바이너리가 Fortran으로 작성되었으며, `libgfortran.so.5`에 의존하고 있음을 확인했습니다. 또한 `strings` 출력에서 `Enter the flag :` 및 `you got itWrong`과 같은 문자열을 통해 플래그 검증 프로그램임을 추정했습니다.

```bash
ls -la /sandbox/bins/
file /sandbox/bins/try
strings /sandbox/bins/try
echo 'AAAA' | /sandbox/bins/try
```

### 2) 정적 분석 도구 활용
Radare2(`r2`)를 사용하여 함수 목록(`afl`)과 `main` 함수의 디스어셈블리 코드를 확인했습니다. `main`은 `dbg.question` 함수를 호출하며, 이 함수가 핵심 검증 로직을 담당하고 있음을 파악했습니다.

```bash
r2 -q "afl; pdf @main" /sandbox/bins/try
```

---

## 🧠 취약점 분석

`dbg.question` 함수를 디컴파일하여 분석한 결과, 다음과 같은 검증 로직이 존재함을 확인했습니다:

- 18바이트의 입력을 받음
- 입력 각 바이트를 고정된 키(`QWERTYUIOPASDFGHJK`)와 XOR
- 짝수 인덱스는 `<< 4`, 홀수 인덱스는 `* 4` 연산 수행
- 결과 값을 `.rodata` 섹션의 `A.0.3887` 심볼에 저장된 값들과 비교
- 모든 비교가 일치하면 `f == 0x12`이 되어 성공 메시지 출력

디컴파일된 코드 일부:
```c
f = 0;
var_30h = 0x4955595452455751;  // "QWERTYUI"
var_28h = 0x484746445341504f;  // "OPASDFGH"
var_20h._0_2_ = 0x4b4a;        // "JK"
// ...
for (n = 1; n <= 0x13; n++) {
    num2 = (&var_31h)[n] ^ (&var_30h)[n];
    if ((n - 1) % 2 == 1) {
        num1 = num2 * 4;
    } else {
        num1 = num2 << 4;
    }
    if (num1 == (&_num)[n - 1]) f++;
}
```

검증 로직은 단순한 수치 변환으로, 역산이 가능합니다. `.rodata` 섹션의 주소 `0x2060`에서 18개의 32비트 리틀엔디언 값이 `A.0.3887` 심볼로 저장되어 있음을 확인했습니다.

---

## 💥 익스플로잇

검증 로직을 역산하기 위해 Python 스크립트를 작성했습니다.  
1. 바이너리에서 `0x2060` 주소부터 72바이트를 읽어 18개의 32비트 정수로 해석  
2. 각 인덱스에 대해:
   - 짝수 인덱스: `>> 4` 후 키와 XOR
   - 홀수 인덱스: `// 4` 후 키와 XOR
3. 결과를 문자열로 조합

```python
BINARY = '/sandbox/bins/try'

def solve():
    with open(BINARY, 'rb') as f:
        f.seek(0x2060)
        expected_bytes = f.read(72)  # 18 * 4 bytes
    import struct
    expected = struct.unpack('<18I', expected_bytes)
    
    key_str = 'QWERTYUIOPASDFGHJK'
    flag = ''
    
    for i in range(18):
        exp_val = expected[i]
        key_byte = ord(key_str[i])
        
        if i % 2 == 0:  # even: was << 4
            if exp_val & 0xF != 0:
                print(f"Invalid at {i}: {exp_val}")
                return
            temp = exp_val >> 4
        else:  # odd: was * 4
            if exp_val % 4 != 0:
                print(f"Invalid at {i}: {exp_val}")
                return
            temp = exp_val // 4
        
        input_byte = temp ^ key_byte
        flag += chr(input_byte)
    
    print('Flag:', flag)

solve()
```

실행 결과:
```
Flag: q214cd644cb1
```

플래그 포맷이 `GLUG{}`임을 유추하여 최종 플래그를 제출했습니다.

---

## 🚩 Flag

```
GLUG{q214cd644cb1}