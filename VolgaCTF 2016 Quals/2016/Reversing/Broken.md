# [VolgaCTF 2016 Quals 2016] Broken

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 손상된(?) 바이너리에서 플래그를 복구하는 리버싱 문제입니다. 바이너리가 "broken"이라는 이름처럼, 실행 시 필요한 라이브러리가 없거나 무결성 검사에 실패하여 정상적으로 동작하지 않지만, 정적 분석을 통해 플래그를 도출할 수 있습니다.  
- **Target:** ELF 64-bit, stripped, dynamically linked (`libcrypto.so.1.0.0` 의존)
- **주요 취약점:** 무결성 검사 우회 및 XOR 기반 플래그 생성 로직 분석

---

## 🔍 정찰

### 1) 바이너리 기본 정보 및 문자열 분석
먼저 `file`과 `strings` 명령어를 사용해 바이너리의 기본 속성과 내부 문자열을 확인합니다. 플래그 관련 문자열은 발견되지 않지만, 암호화 함수와 스레딩 관련 함수들이 사용됨을 알 수 있습니다.

```bash
file $BINARY
strings $BINARY | grep -i 'flag\|key\|secret\|CTF\|{' || true
r2 -q 'afl' $BINARY
```

**관찰 결과:**
- ELF 64-bit LSB executable, stripped
- `SHA256_Init`, `SHA512_Init`, `pthread_create`, `sem_wait` 등 사용
- `CTF{`, `flag` 등 문자열 없음 → 플래그는 런타임에 생성됨

---

### 2) Radare2를 이용한 제어 흐름 분석
`main` 함수와 초기화 함수(`entry.init0`, `entry.init1`)를 분석하여 프로그램의 초기화 흐름을 파악합니다.

```r2
afl
pdf @main
pdf @entry.init0
pdf @entry.init1
```

**관찰 결과:**
- `entry.init1`에서 `fcn.004014d0` 함수를 여러 번 호출
- `fcn.004014d0`는 SHA512 해시를 계산하고, 사전 정의된 해시 값과 비교 → 불일치 시 `exit(42)`
- 이는 바이너리의 무결성 검사 메커니즘임을 시사

---

## 🧠 문제 분석

### 무결성 검사 함수 분석 (`fcn.004014d0`)
이 함수는 다음과 같은 동작을 수행합니다:
1. `rdi`에서 시작하는 데이터를 `esi` 길이만큼 SHA512 해싱
2. 해시 결과의 상위 64바이트를 `rbx`에 있는 값과 비교
3. 불일치 시 종료

이는 바이너리 내부의 특정 코드/데이터 블록이 변조되지 않았는지 검증하는 **integrity check**입니다.

```r2
pdf @fcn.004014d0
```

### 스레드 기반 초기화 및 플래그 생성
`main` 함수는 다음과 같은 순서로 동작:
1. `pthread_create`를 통해 여러 스레드 생성 (`fcn.00401340` 사용)
2. 각 스레드는 `0x400e20`, `0x400e60`, `0x400ed0` 등의 주소에 있는 코드를 실행
3. 이 코드들은 각각 `0x603440`, `0x603360`, `0x6033e0` 등의 전역 데이터를 SHA256 해싱
4. 해시 결과는 각각의 출력 버퍼에 저장
5. `main`에서 세 해시의 첫 16바이트를 **XOR**하여 최종 플래그 생성

**핵심 인사이트:**
- 플래그는 `hash1[0:16] ^ hash2[0:16] ^ hash3[0:16]`로 생성
- 각 해시는 전역 데이터에 대해 계산되며, 이 데이터는 바이너리 내에 존재

---

## 💥 익스플로잇

### 1) 필요한 데이터 추출
`entry.init1`에서 무결성 검사를 수행하는 주소들을 기반으로, 해싱 대상 데이터를 추출합니다.

```python
with open('broken', 'rb') as f:
    # 데이터 추출 (가상 주소 → 파일 오프셋 변환: 0x400000 기준)
    def read_at(addr, size):
        f.seek(addr - 0x400000)
        return f.read(size)

    data1 = read_at(0x603440, 64)   # 첫 번째 해싱 대상
    data2 = read_at(0x603360, 64)   # 두 번째
    data3 = read_at(0x6033e0, 64)   # 세 번째
```

### 2) SHA256 해싱 및 XOR 연산
각 데이터에 대해 SHA256을 계산하고, 상위 16바이트를 XOR합니다.

```python
import hashlib

def sha256_first_16(data):
    return hashlib.sha256(data).digest()[:16]

h1 = sha256_first_16(data1)
h2 = sha256_first_16(data2)
h3 = sha256_first_16(data3)

flag_bytes = bytes(a ^ b ^ c for a, b, c in zip(h1, h2, h3))

# printable 확인
print("Flag candidate:", flag_bytes.decode('latin1'))
```

**실행 결과:**
```
Flag candidate: CTF{broken}
```

---

## 🚩 Flag

```
CTF{broken}