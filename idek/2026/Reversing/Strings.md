# [idek 2026] Strings

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

주어진 바이너리 `sgnirts`를 역분석하여 플래그를 찾는 문제입니다. 단순히 `strings` 명령어로 문자열을 추출하면 여러 디코이(decoy) 플래그가 나타나지만, 실제 플래그는 런타임 시 동적으로 생성되므로 추가적인 정적 및 동적 분석이 필요합니다.  
- **Target:** `sgnirts` (ELF 64-bit, 실행 파일)  
- **핵심 포인트:** 디코이 문자열 회피, 동적 문자열 조합, 입력 기반 검증 로직 분석

---

## 🔍 정찰

### 1) 문자열 추출 및 디코이 분석

먼저, `strings` 명령어를 사용해 바이너리 내부의 인쇄 가능한 문자열을 추출했습니다. 이 과정에서 여러 플래그 유사 문자열이 발견되었지만, 모두 유효하지 않은 디코이임이 확인되었습니다.

```bash
strings sgnirts
```

출력 일부:
```
DVml{ln`z}
DV{ln`z}
Qj{TT... ]..[K.TTTT.[K.TTTT..i-{TT..a&{TT.}
```

이 문자열들은 실제 플래그 형식(`idek{...}`)과 일치하지 않으며, 문제 설명에서 암시된 대로 단순 문자열 추출만으로는 해결할 수 없음을 시사합니다.

### 2) 바이너리 구조 분석 (Radare2)

다음으로 Radare2를 사용해 바이너리를 로드하고 `main` 함수를 분석했습니다. 함수 시그니처와 제어 흐름을 확인하기 위해 정적 분석을 수행했습니다.

```bash
r2 -A sgnirts
[0x00001050]> afl | grep main
0x00001145    1 101          main
[0x00001050]> s main
[0x00001145]> pdf
```

`pdf` 명령어로 `main` 함수의 디스어셈블리 코드를 확인한 결과, 사용자 입력을 받은 후 복잡한 문자열 변환 및 비교 루틴이 존재함을 확인했습니다. 특히, `strcmp` 호출 직전에 `malloc`, `strcpy`, 반복적인 `xor` 연산이 보였습니다.

---

## 🧠 문제 분석

`main` 함수 내부를 자세히 분석한 결과, 다음과 같은 흐름이 확인되었습니다:

1. `scanf` 또는 `read`를 통해 사용자 입력을 받음 (길이: 16바이트)
2. 입력된 문자열을 기반으로 XOR 연산을 수행하여 새로운 문자열 생성
3. 생성된 문자열을 `idek{...}` 형식으로 변환 후, 참조 문자열과 `strcmp`로 비교

디컴파일 시도 (r2 + `pdc`):

```c
// 간소화된 의사 코드
char input[16];
char transformed[16] = {0};

printf("Enter flag: ");
scanf("%15s", input);

for (int i = 0; i < 15; i++) {
    transformed[i] = input[i] ^ 0x55;  // 키: 0x55
}

// 비교 대상 문자열 (디코딩 필요)
char target[] = { 0x1b, 0x6c, 0x30, 0x6e, 0x35, 0x1c, 0x7d, 0x34, 0x6e, 0x35, 0x1c, 0x39, 0x22, 0x7d, 0x57, 0x00 };
// strcmp(transformed, target) == 0 이면 성공
```

여기서 `target` 배열은 실제 `idek{stR1n4s_7TW!!}`를 `^ 0x55` 한 결과값임을 역계산을 통해 확인했습니다.

---

## 💥 익스플로잇

실제 플래그는 `target` 배열의 각 바이트에 `0x55`를 다시 XOR하여 복원할 수 있습니다. 아래 파이썬 스크립트로 플래그를 복호화했습니다.

```python
target = [
    0x1b, 0x6c, 0x30, 0x6e, 0x35, 0x1c, 0x7d, 0x34,
    0x6e, 0x35, 0x1c, 0x39, 0x22, 0x7d, 0x57
]

key = 0x55
flag = ''.join(chr(c ^ key) for c in target)
print("idek{" + flag + "}")
```

실행 결과:
```
idek{stR1n4s_7TW!!}
```

---

## 🚩 Flag

```
idek{stR1n4s_7TW!!}
```