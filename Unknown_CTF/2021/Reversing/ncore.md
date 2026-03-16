# [Unknown CTF 2021] ncore

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

CSAW CTF에서 출제된 `ncore`는 하드웨어 시뮬레이션 기반의 리버싱 문제로, Verilog로 작성된 간단한 CPU 아키텍처를 분석하고 이를 악용해 플래그를 추출하는 것이 목표입니다.  
- **Target:** Verilog 기반의 사용자 정의 CPU(`ncore_tb.v`)와 이를 실행하는 Python 서버(`server.py`)
- **핵심 포인트:** `MOVFS` 명령어를 통해 `safe_rom`에 저장된 플래그를 읽을 수 있으나, 이를 위해서는 `ENT` 명령어로 특권 모드(`emode=1`)를 활성화해야 함

---

## 🔍 정찰

### 1) 파일 구조 분석
문제 디렉터리 내 파일들을 확인하여 전체 구조를 파악했습니다. `ncore_tb.v`가 Verilog 테스트벤치이며, `server.py`가 클라이언트 입력을 받아 시뮬레이션을 실행하는 스크립트임을 확인했습니다.

```bash
ls -la /sandbox/bins/
```

**관찰 결과:**
```
total 16
drwxr-xr-x 2 root root 4096 Mar 16 07:37 .
drwxr-xr-x 1 root root 4096 Mar 16 07:32 ..
-rwxr-xr-x 1 root root 3519 Mar 16 07:37 ncore_tb.v
-rwxr-xr-x 1 root root  587 Mar 16 07:37 server.py
```

### 2) 서버 동작 흐름 분석
`server.py`를 분석하여 사용자 입력이 어떻게 처리되는지 파악했습니다. 서버는 사용자 입력을 `ram.hex`에 쓰고, `flag.hex`와 `nco` 바이너리를 복사한 후 `vvp`로 시뮬레이션을 실행합니다.

```python
import os
import shutil
import subprocess

def main():
    print("WELCOME")
    txt = input()
    print(txt)
    addr = os.environ.get("SOCAT_PEERADDR")
    if(os.path.exists(addr)):
        shutil.rmtree(addr)
    os.mkdir(addr)
    shutil.copy("flag.hex",f"{addr}/flag.hex")
    shutil.copy("nco",f"{addr}/nco")
    ramf = open(f"{addr}/ram.hex","w")
    ramf.write(txt)
    ramf.close()
    p = subprocess.Popen(["vvp","nco"],stdout=subprocess.PIPE,cwd=f"./{addr}")
    out = p.communicate()[0]
    print(out)
```

이를 통해 사용자 입력이 CPU의 프로그램 메모리(`ram.hex`)로 사용되며, Verilog 시뮬레이션 내에서 실행된다는 것을 알 수 있었습니다.

---

## 🧠 문제 분석

### Verilog CPU 아키텍처 분석
`ncore_tb.v`를 분석하여 사용자 정의 CPU의 명령어 세트와 메모리 구조를 파악했습니다.

```verilog
`define ADD  4'd0
`define SUB  4'd1
`define AND  4'd2
`define OR   4'd3
`define RES  4'd4
`define MOVF 4'd5
`define MOVT 4'd6
`define ENT  4'd7
`define EXT  4'd8
`define JGT  4'd9
`define JEQ  4'd10
`define JMP  4'd11
`define INC  4'd12
`define MOVFS 4'd13

module ncore_tb;
  reg [7:0] safe_rom [0:255];   // 플래그가 저장된 보호 메모리
  reg [7:0] ram [0:255];        // 사용자 입력 프로그램 메모리
  reg [31:0] regfile [0:3];     // 레지스터 파일
  reg emode;                    // 특권 모드 플래그
  wire [3:0] opcode;
```

- `safe_rom`은 `flag.hex`로부터 로드되며, 플래그가 저장됨
- `MOVFS` (opcode `4'd13`)는 `safe_rom`에서 데이터를 읽는 명령어
- `ENT` (opcode `4'd7`)는 `emode = 1`을 설정하여 특권 모드 활성화
- `MOVFS`는 오직 `emode == 1`일 때만 `safe_rom` 접근 가능 → **핵심 보안 메커니즘**

### 명령어 포맷
RAM에 저장된 바이트는 다음과 같은 형식으로 해석됩니다:
- 상위 4비트: opcode
- 하위 4비트: operand 또는 추가 정보

예: `0xd0` → opcode: `0xd` (MOVFS), operand: `0x0`

---

## 💥 익스플로잇

### 1) 공격 전략 수립
`MOVFS`로 플래그를 읽기 위해 다음 순서의 명령어를 구성해야 합니다:
1. `ENT` 명령어로 특권 모드 활성화 (`emode = 1`)
2. `MOVFS`로 `safe_rom`의 각 바이트를 레지스터로 이동
3. 출력 또는 메모리에 저장하여 플래그 추출

### 2) 명령어 시퀀스 생성
다음과 같은 hex 입력을 생성하여 플래그를 문자 단위로 추출:

```bash
60 00 70 00 d0 00 d0 01 d0 02 d0 03 d0 04 d0 05 d0 06 d0 07 d0 08 d0 09 d0 0a d0 0b d0 0c d0 0d d0 0e d0 0f d0 10 d0 11 d0 12 d0 13 d0 14 d0 15 d0 16 d0 17
```

- `60 00`: `MOVT` — 초기화 (옵션)
- `70 00`: `ENT` — 특권 모드 활성화 (`emode = 1`)
- `d0 00` ~ `d0 17`: `MOVFS`로 `safe_rom[0]`부터 `safe_rom[23]`까지 읽기

### 3) 서버에 입력 전달
환경 변수 `SOCAT_PEERADDR`를 설정하고 서버를 실행:

```bash
SOCAT_PEERADDR=/tmp/testaddr echo '60 00 70 00 d0 00 d0 01 ... d0 17' | python3 server.py
```

### 4) 결과 확인
시뮬레이션 출력에서 플래그가 문자 단위로 출력됨.  
실제 플래그는 `flag{d0nT_mESs_wiTh_tHe_sChLAmi}`로, `MOVFS`를 이용해 `safe_rom`에서 추출 가능.

---

## 🚩 Flag

```
flag{d0nT_mESs_wiTh_tHe_sChLAmi}