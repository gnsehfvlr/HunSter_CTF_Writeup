# [dvCTF 2026] rocca-pia

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 주어진 ELF 바이너리를 역분석하여 올바른 입력을 찾아 플래그를 출력하게 만드는 전형적인 리버싱 챌린지입니다.  
프로그램은 사용자 입력을 특정 알고리즘으로 변환한 후, 정해진 값과 비교하며, 이 검증을 통과하면 플래그를 출력합니다.

- **Target:** `rocca_pia` (ELF 64-bit, dynamically linked)
- **주요 취약점:** 입력값에 대한 단순한 XOR 기반 변환 및 정적 문자열 비교로 인해 역연산을 통해 플래그를 복원할 수 있음

---

## 🔍 정찰

### 1) 초기 정찰 및 바이너리 분석

문제를 해결하기 위해 먼저 바이너리의 기본 정보를 확인하고, 포함된 문자열과 동작 방식을 분석합니다.  
`file`, `strings`, 그리고 간단한 실행 테스트를 통해 프로그램의 동작을 파악합니다.

```bash
ls -la /sandbox/bins/ && \
echo "\n[File info]" && \
file $BINARY && \
echo "\n[Strings]" && \
strings $BINARY && \
echo "\n[Run test]" && \
$BINARY "AAAA" || echo "(binary exited)"
```

**관찰 결과:**
- 바이너리는 64비트 ELF이며, 동적 링크 방식 사용
- `Usage: %s <password>`, `Nice try`, `Nice flag`, `PASSWD` 등의 문자열 존재
- 임의의 입력으로 실행 시 세그폴트 발생 → 입력 형식이 특정 조건을 만족해야 함
- `transform` 함수와 `strncmp` 호출이 눈에 띔 → 입력 변환 후 비교 로직 존재

---

### 2) main 함수 분석을 통한 흐름 파악

`main` 함수를 디컴파일하여 프로그램의 실행 흐름을 확인합니다.  
입력 인자가 2개일 때(즉, 실행 파일명 + 비밀번호) `transform` 함수를 호출하고, 그 반환값에 따라 출력을 결정합니다.

```c
ulong main(uint32_t argc, char **argv) {
    int iVar1;
    ulong uVar2;
    if (argc == 2) {
        iVar1 = sym.transform(argv[1]);
        if (iVar1 == 0) {
            sym.imp.puts(0x2026);  // "Nice flag" 출력
        } else {
            sym.imp.puts("Nice try");
        }
        uVar2 = 0;
    } else {
        sym.imp.printf("Usage: %s <password>\n");
        uVar2 = 1;
    }
    return uVar2;
}
```

- `transform` 함수가 0을 반환하면 `0x2026` 주소의 문자열(`Nice flag`) 출력
- `transform` 함수의 로직이 핵심임을 확인

---

## 🧠 문제 분석

### transform 함수의 어셈블리 분석

`transform` 함수의 디컴파일이 실패했으므로, `radare2`를 사용해 어셈블리 코드를 직접 분석합니다.

```bash
r2 -q -c "pdf @sym.transform" /sandbox/bins/rocca_pia
```

**핵심 로직 추출:**
1. 입력 문자열의 각 바이트를 인덱스에 따라 다르게 XOR 처리:
   - 짝수 인덱스: `byte ^ 0x13`
   - 홀수 인덱스: `byte ^ 0x37`
2. 변환된 결과를 버퍼에 저장
3. 이 결과를 정적 문자열과 `strncmp`로 비교 (비교 길이: `0x16` = 22바이트)

또한, `strings` 출력에서 `"wAPcULZh"`가 발견되었고, 이는 비교 문자열의 일부임을 시사합니다.  
또한 `0x2010` 주소에 `obj.PASSWD`가 위치하며, 이곳에 비교 대상 데이터가 저장되어 있음.

---

### 정적 비교 데이터 확인

`0x2010` 주소에서 22바이트를 hex 덤프하여 전체 비교 문자열을 추출합니다.

```bash
# read_file 대신 hexdump 사용 (동등한 동작)
xxd -s 8208 -l 32 /sandbox/bins/rocca_pia
```

**출력 일부:**
```
00002010: 7741 5063 554c 5a68 7f06 7804 4c44 6406  wAPcULZh..x.LDd.
00002020: 7e5a 2259 744a 4e69 6365 2066 6c61 6700  ~Z"YtJNice flag.
```

- `0x2010`부터 22바이트: `77 41 50 63 55 4c 5a 68 7f 06 78 04 4c 44 64 06 7e 5a 22 59 74 4a`
- 이 값은 `strncmp`의 비교 대상 (`s2` vs `obj.PASSWD`)임
- 즉, 입력을 XOR 변환한 결과가 이 22바이트와 일치해야 성공

---

## 💥 익스플로잇

### XOR 변환 역연산을 통한 플래그 복원

입력 → XOR 변환 → 비교  
따라서, 비교 문자열 → 역 XOR → 원래 입력(플래그) 복원 가능

- 짝수 인덱스: `c ^ 0x13`
- 홀수 인덱스: `c ^ 0x37`

```python
# Full 22-byte expected transformed data from memory
expected = bytes.fromhex('77415063554c5a687f0678044c4464067e5a2259744a')

# Reverse the XOR: even indices XOR 0x13, odd indices XOR 0x37
password = bytearray()
for i, c in enumerate(expected):
    if i % 2 == 0:
        password.append(c ^ 0x13)
    else:
        password.append(c ^ 0x37)

print(password.decode('ascii'))
```

**실행 결과:**
```
dvCTF{I_l1k3_sw1mm1ng}
```

이 값이 정확히 22바이트이며, `{`와 `}` 포함, 플래그 형식을 완전히 만족함.

---

## 🚩 Flag

```
dvCTF{I_l1k3_sw1mm1ng}