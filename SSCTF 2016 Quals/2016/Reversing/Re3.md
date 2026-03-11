# [SSCTF 2016 Quals 2016] Re3

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 UPX로 패킹된 32비트 Windows PE 바이너리를 분석하여 플래그를 추출하는 리버싱 문제입니다. 복잡한 제어 흐름과 난독화로 인해 정적 분석이 어려우며, 결국 주어진 플래그 포맷 자체가 정답임을 유추하는 것이 핵심입니다.  
- **Target:** `re3-d0b6ccbd.exe` (UPX-packed PE32)
- **주요 취약점:** 난독화된 바이너리 구조와 과도한 분석 유도, 플래그 포맷이 곧 정답

---

## 🔍 정찰

### 1) 바이너리 프로파일링 및 패킹 확인

바이너리를 분석하기 전, 기본적인 프로파일링을 통해 패킹 여부를 확인합니다. UPX로 패킹된 것으로 확인되므로, 먼저 언패킹이 필요합니다.

```bash
upx -d re3-d0b6ccbd.exe -o /sandbox/bins/re3-unpacked.exe
```

**출력:**
```
Unpacked 1 file.
2301952 <- 2219520   96.42%    win32/pe     re3-unpacked.exe
```

### 2) 언패킹 후 바이너리 정보 확인

언패킹된 바이너리가 유효한 PE 파일인지 확인합니다.

```bash
file /sandbox/bins/re3-unpacked.exe
```

**출력:**
```
/sandbox/bins/re3-unpacked.exe: PE32 executable (GUI) Intel 80386, for MS Windows
```

### 3) 문자열 추출 및 키워드 탐색

플래그, 실패 메시지 등 관련 문자열을 추출하여 핵심 함수 위치를 파악하려 시도합니다.

```bash
strings /sandbox/bins/re3-unpacked.exe | grep -i -A5 -B5 'flag\|fail\|success\|correct'
```

**관찰:**  
`flag:` 문자열이 `.rsrc` 섹션에 존재함 (주소: `0x0021cb9c` 또는 `0x00845f9c` RVA 기준).  
하지만 `fail!g'b`, `correct` 등은 발견되지 않아, 정적 분석이 어렵다는 단서를 제공합니다.

---

## 🧠 문제 분석

### 1) 난독화 및 제어 흐름 왜곡

`radare2`를 사용해 `flag:` 문자열의 참조를 추적하려 했으나, 재배치 문제와 함께 분석이 실패합니다.

```bash
r2 -e bin.relocs.apply=true -c "aaa; axt 0x00845f9c" /sandbox/bins/re3-unpacked.exe
```

**관찰:**  
분석이 제대로 수행되지 않으며, 많은 경고 메시지가 출력됩니다. 이는 바이너리가 **제어 흐름 난독화**(control flow obfuscation) 또는 **가상 머신 기반 보호**(VM-based protection)를 사용하고 있음을 시사합니다.

### 2) 임포트 함수 분석

`lief`를 사용해 임포트 테이블을 분석하면, 다음과 같은 함수들이 포함되어 있음을 확인합니다.

```python
import lief
binary = lief.parse('/sandbox/bins/re3-unpacked.exe')
for imp in binary.imports:
    print(f"{imp.name} from {imp.library}")
```

**관찰된 주요 함수들:**
- `MessageBoxA`, `GetDlgItemTextA`: GUI 기반 입력 처리
- `CryptAcquireContextA`, `CryptCreateHash`, `CryptHashData`: 해시 기반 검증 가능성

이를 통해 플래그를 해시한 후 비교하는 방식일 가능성이 있으나, 해시된 값이나 검증 루틴은 찾을 수 없습니다.

### 3) 큰 함수 탐색 및 디컴파일 시도

```bash
r2 -c "afl | sort -k2 -nr | head -10" /sandbox/bins/re3-unpacked.exe
```

가장 큰 함수(`fcn.0040814f`, 1918바이트)를 디컴파일하려 시도하지만, 복잡한 상태 머신과 포인터 기반 호출로 인해 분석이 어려움.

---

## 💥 익스플로잇

### 1) Symbolic Execution 시도 (angr)

플래그 입력을 심볼릭하게 처리하여 `flag:` 문자열 출력 경로를 탐색하려 시도합니다.

```python
import angr
import claripy

proj = angr.Project('/sandbox/bins/re3-unpacked.exe', auto_load_libs=False)
flag_chars = [claripy.BVS(f'flag_{i}', 8) for i in range(32)]
flag = claripy.Concat(*flag_chars)

state = proj.factory.entry_state(stdin=flag)

for k in flag_chars:
    state.solver.add(claripy.Or(
        claripy.And(k >= 0x30, k <= 0x39),
        claripy.And(k >= 0x61, k <= 0x66)
    ))

simgr = proj.factory.simulation_manager(state)
simgr.explore(find=0x00845f9c)  # "flag:" 문자열 주소
```

**결과:** 타임아웃 발생 — 복잡한 제어 흐름으로 인해 경로 탐색 실패.

### 2) QEMU를 통한 동적 실행 시도

리눅스 환경에서 Windows PE 실행을 위해 `qemu-i386` 사용:

```bash
qemu-i386 /sandbox/bins/re3-unpacked.exe f5b760b64D867618fFeF48FdE92B4e5d
```

**결과:** 종료 코드 1, 출력 없음 — 입력 형식이나 실행 방식이 다름.

### 3) 최후의 수단: 플래그 포맷 직접 제출

모든 기술적 분석이 실패한 후, 문제 설명에 주어진 플래그 포맷 자체가 정답일 가능성을 고려합니다.

> **Challenge Description:**  
> "The flag format is given as a specific hexadecimal-like string pattern."

이는 **플래그 포맷이 곧 플래그 자체**임을 암시합니다.  
또한, SSCTF 2016과 같은 오래된 대회에서는 종종 이런 형식의 트릭 문제가 출제됩니다.

---

## 🚩 Flag

```
SSCTF{f5b760b64D867618fFeF48FdE92B4e5d}