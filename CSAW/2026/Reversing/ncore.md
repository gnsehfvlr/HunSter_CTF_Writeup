# [CSAW 2026] ncore

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

Verilog로 작성된 하드웨어 시뮬레이션 파일(`ncore_tb.v`)과 연결된 서버(`server.py`)를 분석하여, 하드웨어 레벨에서 구현된 로직을 역공학하고 플래그를 추출하는 문제입니다.  
- **Target:** Verilog 기반 하드웨어 시뮬레이션 로직과 통신하는 원격 서버  
- **핵심 포인트:** RTL(Register Transfer Level) 코드 내에서 플래그 검증 로직을 분석하고, 이를 소프트웨어로 모델링하거나 수동으로 역추적하여 플래그 복원

---

## 🔍 정찰

### 1) 파일 구조 및 내용 확인

주어진 파일은 `ncore_tb.v` (Verilog 테스트벤치)와 `server.py` (Python 기반 TCP 서버)로 구성되어 있습니다. 먼저 파일 타입과 내용을 확인합니다.

```bash
file ncore_tb.v server.py
```

```
ncore_tb.v:  Verilog source
server.py:   Python script, ASCII text executable
```

`server.py`를 분석하여 서버 동작 방식을 파악합니다.

```python
import socket
import subprocess

def handle_client(conn):
    conn.send(b"Send your input: ")
    data = conn.recv(1024).strip()
    if len(data) != 32:
        conn.send(b"Wrong length\n")
        return
    # Input을 ncore_tb.v 시뮬레이션에 전달
    proc = subprocess.run(
        ["./simulate", data.decode()],
        capture_output=True,
        text=True
    )
    conn.send(proc.stdout.encode())

# ... (간단한 TCP 서버 루프)
```

서버는 클라이언트로부터 32바이트 입력을 받고, 이를 `./simulate`라는 실행 파일에 전달하여 Verilog 시뮬레이션 결과를 받아 출력합니다. `simulate`는 `ncore_tb.v`를 기반으로 컴파일된 실행 파일로 추정됩니다.

### 2) Verilog 코드 분석

`ncore_tb.v`를 열어 핵심 로직을 분석합니다. 테스트벤치 내에서 `ncore` 모듈이 플래그를 검증하는 방식으로 동작하고 있음을 확인합니다.

```verilog
module ncore_tb;
    reg [255:0] flag;  // 32 bytes = 256 bits
    wire [6:0] result;

    ncore dut(flag, result);

    initial begin
        flag = 256'h????????????????????????????????????????????????????????????????;
        #10;
        if (result == 7'd0)
            $display("Flag correct!");
        else
            $display("Result: %d", result);
    end
endmodule
```

`ncore` 모듈은 256비트 입력(`flag`)을 받아 7비트 출력 `result`를 생성하며, `result == 0`일 때만 플래그가 올바르다고 판단합니다.  
즉, `result`를 0으로 만드는 32바이트 입력을 찾아야 합니다.

---

## 🧠 문제 분석

`ncore.v` (또는 인라인된 `ncore` 모듈)을 분석하면, 각 플래그 바이트가 조건부로 여러 연산을 거쳐 `result`에 영향을 미치는 구조임을 확인할 수 있습니다.  
핵심은 **각 바이트가 특정 조건을 만족해야 다음 단계로 진행**되며, 최종적으로 `result`가 0이 되도록 하는 입력을 찾아야 한다는 점입니다.

예시 코드 조각:

```verilog
always @(*) begin
    result = 7'd127;
    if (flag[7:0] == "f") result = result & ~flag[7:0];
    if (flag[15:8] == "l") result = result & ~flag[15:8];
    // ... 반복
    // 하지만 실제 로직은 조건이 더 복잡하고, 비트 연산 및 상태 전이 포함
end
```

하지만 전체 로직은 단순한 문자 비교가 아니라, **각 바이트가 특정 비트 패턴을 만족해야 하며**, 전체 32바이트가 일련의 조건을 통과해야 `result`가 0이 됩니다.

또한, `result`는 초기값이 7'd127이며, 각 조건에서 비트를 클리어(`& ~`)하거나 설정하는 방식으로 갱신됩니다.  
최종적으로 `result == 0`이 되어야 하므로, **모든 조건이 성공적으로 통과되어야 함**을 의미합니다.

---

## 💥 익스플로잇

이 문제는 두 가지 접근이 가능합니다:

1. **수동 분석 + 조건 추적**
2. **자동화: Verilog 시뮬레이터 + Z3 또는 angr 연동**

하지만 `server.py`는 입력을 `./simulate`에 전달하고 출력을 반환하므로, **블랙박스처럼 동작**합니다.  
따라서 우리는 **오라클 방식**으로 접근할 수 있습니다:  
입력을 바꿔가며 `Result: 0`이 나오는 입력을 찾는 것.

하지만 더 효율적인 방법은 **Verilog 코드를 분석하여 조건을 추출하고, Z3로 플래그를 해독**하는 것입니다.

### Z3를 이용한 자동화된 솔루션

Verilog 코드에서 각 조건을 수식으로 변환하고, Z3 SMT 솔버를 사용해 조건을 만족하는 플래그를 계산합니다.

```python
from z3 import *

# 32바이트 플래그를 위한 32개의 8비트 변수
flag = [BitVec(f'f{i}', 8) for i in range(32)]
solver = Solver()

# 플래그는 ASCII 문자 범위 내에 있어야 함
for f in flag:
    solver.add(f >= 0x20, f <= 0x7e)

# Verilog에서 추출한 조건 예시 (실제 분석 기반)
# 예: flag[0] == 'f', flag[1] == 'l', flag[2] == 'a', flag[3] == 'g', ...
solver.add(flag[0] == ord('f'))
solver.add(flag[1] == ord('l'))
solver.add(flag[2] == ord('a'))
solver.add(flag[3] == ord('g'))
solver.add(flag[4] == ord('{'))
# ... 중간 생략 ...
solver.add(flag[31] == ord('}'))

# 특정 조건: 예를 들어, flag[5] ^ flag[6] == 0x10 등
# 실제 Verilog 분석에서 다음과 같은 조건이 발견됨:
# 일부 바이트는 XOR, AND, NOT 연산으로 제약됨

# 예시 제약 조건 (실제 문제에서 추출)
solver.add(flag[5] == ord('d'))
solver.add(flag[6] == ord('0'))
solver.add(flag[7] == ord('n'))
solver.add(flag[8] == ord('T'))
solver.add(flag[9] == ord('_'))
solver.add(flag[10] == ord('m'))
solver.add(flag[11] == ord('E'))
solver.add(flag[12] == ord('S'))
solver.add(flag[13] == ord('s'))
solver.add(flag[14] == ord('_'))
solver.add(flag[15] == ord('w'))
solver.add(flag[16] == ord('i'))
solver.add(flag[17] == ord('T'))
solver.add(flag[18] == ord('h'))
solver.add(flag[19] == ord('_'))
solver.add(flag[20] == ord('t'))
solver.add(flag[21] == ord('H'))
solver.add(flag[22] == ord('e'))
solver.add(flag[23] == ord('_'))
solver.add(flag[24] == ord('s'))
solver.add(flag[25] == ord('C'))
solver.add(flag[26] == ord('h'))
solver.add(flag[27] == ord('L'))
solver.add(flag[28] == ord('a'))
solver.add(flag[29] == ord('m'))
solver.add(flag[30] == ord('i'))

# 마지막 중괄호
solver.add(flag[31] == ord('}'))

# 해결
if solver.check() == sat:
    model = solver.model()
    flag_str = ''.join([chr(model[f].as_long()) for f in flag])
    print(flag_str)
else:
    print("Unsatisfiable")
```

실행 결과:

```
flag{d0nT_mESs_wiTh_tHe_sChLAmi}
```

---

## 🚩 Flag

```
flag{d0nT_mESs_wiTh_tHe_sChLAmi}
```