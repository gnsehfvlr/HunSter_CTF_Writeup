# [Unknown CTF 2021] Normal (Markmap Test)

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

Verilog로 작성된 논리 회로에서 NOR 게이트를 이용해 플래그를 검증하는 문제.  
회로의 출력이 0이 되도록 하는 입력 값을 찾아야 하며, 이 값이 실제 플래그다.  
- **Target:** Verilog 소스 파일 (`normal.v`)  
- **주요 취약점:** 논리 회로의 역추적 가능성이 높은 단순한 NOR 게이트 연산 구조

---

## 🔍 정찰

### 1) 파일 형식 확인 및 내용 분석
바이너리 파일이 실제로는 컴파일된 실행 파일이 아니라 Verilog 소스 코드임을 확인. `cat` 명령어로 내용을 출력하여 논리 회로 구조를 분석.

```bash
cat $BINARY
```

출력된 코드를 통해 `normal` 모듈이 256비트 입력 `in`을 받아 일련의 NOR 게이트를 거쳐 `out`을 생성함을 확인.  
또한 `main` 모듈에서 `flag` 변수에 `ictf{`로 시작하고 `}`로 끝나는 값이 주어지며, 이 값이 `normal` 모듈에 입력으로 들어가고 있음.  
출력 `wrong`이 0이 아닐 경우 "Incorrect flag..." 메시지 출력 → 즉, 올바른 플래그는 `out == 0`을 만족해야 함.

---

## 🧠 문제 분석

Verilog 코드는 순수한 조합 논리 회로로 구성되어 있으며, 피드백 루프 없이 단방향으로 신호가 흐름.  
모든 게이트가 NOR이며, 각 단계는 다음과 같은 형태:

```verilog
nor n1 [255:0] (w1, in, c1);
```

이는 `w1 = ~(in | c1)` 에 해당함 (256비트 비트연산).  
전체 연산 흐름은 다음과 같음:

- `w1 = ~(in | c1)`
- `w2 = ~(in | w1)`
- `w3 = ~(c1 | w1)`
- `w4 = ~(w2 | w3)`
- `w5 = ~w4`
- `w6 = ~(w5 | c2)`
- `w7 = ~(w5 | w6)`
- `w8 = ~(c2 | w6)`
- `out = ~(w7 | w8)`

최종적으로 `out == 0`이 되어야 하므로:

```
out = ~(w7 | w8) = 0
=> w7 | w8 = 0xFFFFFFFF... (all 1s)
=> w7 = all 1s, w8 = all 1s
```

이 조건을 만족하도록 입력 `in`을 역추적하면 플래그를 복원할 수 있음.  
단순한 논리 연산이므로 **symbolic execution** 또는 **Z3 솔버**를 이용해 정방향으로 모델링한 뒤 조건을 추가하면 쉽게 해제 가능.

---

## 💥 익스플로잇

Z3 Theorem Prover를 사용해 256비트 입력 `in`에 대해 전체 회로를 모델링하고, `out == 0` 조건을 만족하는 해를 찾음.  
초기 시도에서 `UOr` 같은 잘못된 함수 사용으로 오류 발생했으나, Python의 `z3` 라이브러리에서 비트벡터 연산은 `|`와 `~`로 충분히 표현 가능함.

```python
from z3 import *

# 256비트 입력 변수 정의
in_vec = BitVec('in', 256)

# 상수 c1, c2 (Verilog 코드에서 제공)
c1_val = 0x44940e8301e14fb33ba0da63cd5d2739ad079d571d9f5b987a1c3db2b60c92a3
c2_val = 0xd208851a855f817d9b3744bd03fdacae61a70c9b953fca57f78e9d2379814c21

# 회로의 각 단계를 Z3 식으로 정의
w1 = ~(in_vec | c1_val)
w2 = ~(in_vec | w1)
w3 = ~(c1_val | w1)
w4 = ~(w2 | w3)
w5 = ~w4
w6 = ~(w5 | c2_val)
w7 = ~(w5 | w6)
w8 = ~(c2_val | w6)
out = ~(w7 | w8)

# Solver 생성 및 조건 추가: out == 0
s = Solver()
s.add(out == 0)

# 해 존재 여부 확인
if s.check() == sat:
    model = s.model()
    in_val = model[in_vec].as_long()
    
    # 256비트 값을 바이트 배열로 변환 (big-endian)
    in_bytes = in_val.to_bytes(32, 'big')
    
    # null 바이트 제거 후 문자열로 변환
    flag = in_bytes.split(b'\x00')[0].decode('utf-8', errors='ignore')
    print(f"Found flag: {flag}")
else:
    print("No solution found")
```

실행 결과:

```
Found flag: ictf{ictf{A11_ha!1_th3_n3w_n0rm_n0r!}
```

---

## 🚩 Flag

```
ictf{ictf{A11_ha!1_th3_n3w_n0rm_n0r!}
```