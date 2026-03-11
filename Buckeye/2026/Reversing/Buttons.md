# [Buckeye 2026] Buttons

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

Buttons는 Java 기반 GUI 애플리케이션인 `Buttons.jar`를 역분석하여 숨겨진 플래그를 추출하는 리버싱 문제입니다. 사용자는 버튼을 클릭해 미로를 탐색하는 방식으로 플래그를 획득할 수 있으나, 실제로는 정해진 경로를 따라 이동한 후 그 기록을 기반으로 암호화된 플래그를 복호화하는 로직이 구현되어 있습니다.  
- **Target:** `Buttons.jar` (Java GUI Application)
- **주요 취약점:** 플래그 생성 로직이 클라이언트 측에서 완전히 구현되어 있으며, 이는 역공학을 통해 직접 계산 가능함.

---

## 🔍 정찰

### 1) 파일 구조 확인 및 JAR 분석
문제에서 제공된 `/sandbox/bins/` 디렉터리 내 파일을 확인한 결과, `Buttons.jar` 하나만 존재합니다. `file` 명령어를 통해 이 파일이 Java Archive임을 확인하고, `jar` 명령어로 내부를 추출합니다.

```bash
ls -la /sandbox/bins/
```
```
total 16
drwxr-xr-x 2 root root 4096 Mar 10 15:29 .
drwxr-xr-x 1 root root 4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root 4194 Mar 10 15:29 Buttons.jar
```

```bash
file /sandbox/bins/Buttons.jar
```
```
/sandbox/bins/Buttons.jar: Java archive data (JAR)
```

```bash
cd /sandbox/bins && jar -xf Buttons.jar && ls -la
```
```
total 32
drwxr-xr-x 3 root root 4096 Mar 10 15:29 .
drwxr-xr-x 1 root root 4096 Mar 10 13:01 ..
-rw-r--r-- 1 root root 8201 Oct  6  2021 Buttons.class
-rwxr-xr-x 1 root root 4194 Mar 10 15:29 Buttons.jar
drwxr-xr-x 2 root root 4096 Oct  6  2021 META-INF
```

### 2) 클래스 파일 디컴파일 및 소스 코드 분석
`Buttons.class`는 메인 로직을 포함하고 있을 가능성이 높으므로, `jadx`를 사용해 디컴파일합니다. 이후 생성된 출력 디렉터리에서 Java 소스 코드를 확인합니다.

```bash
jadx -d /sandbox/bins/output /sandbox/bins/Buttons.class
```

디컴파일 결과, `/sandbox/bins/output/sources/defpackage/Buttons.java` 경로에 소스 코드가 생성됩니다. `cat` 명령어로 전체 내용을 출력하여 분석합니다.

```bash
cat /sandbox/bins/output/sources/defpackage/Buttons.java
```

---

## 🧠 문제 분석

디컴파일된 `Buttons.java` 코드를 분석하면, 다음과 같은 핵심 로직이 존재합니다:

- 21x21 크기의 그리드 기반 미로가 정의되어 있으며, `0`은 이동 가능한 칸, `1`은 벽입니다.
- 시작 위치는 `(0, 1)`이며, 목표는 `(20, 19)`에 도달하는 것입니다.
- 사용자가 버튼을 통해 이동할 때마다 `moveHistory` 리스트에 `(x * 21 + y)` 형태의 인덱스가 기록됩니다.
- 정답 경로를 완주하면 `printFlag()` 메서드가 호출되며, 이 메서드는 `moveHistory`를 기반으로 플래그를 생성합니다.

`printFlag()` 메서드의 핵심 로직은 다음과 같습니다:

```java
BigInteger result = BigInteger.ONE;
for (int i = 0; i < moveHistory.size(); i++) {
    int val = moveHistory.get(i);
    BigInteger prime = nextProbablePrime(BigInteger.valueOf(val));
    result = result.multiply(prime.pow(i + 1));
}
String flag = new String(result.toByteArray(), StandardCharsets.UTF_8);
JOptionPane.showMessageDialog(this, flag);
```

즉, `moveHistory`의 각 요소 `val`에 대해 `i+1`번째 소수를 구하고, 이를 `(i+1)`제곱하여 누적 곱셈합니다. 최종 `BigInteger` 값을 UTF-8 문자열로 변환하면 플래그가 됩니다.

이 로직은 **완전히 결정론적**이며, GUI를 직접 플레이하지 않고도 올바른 `moveHistory`를 계산하면 플래그를 복원할 수 있습니다.

---

## 💥 익스플로잇

### 1) 미로 경로 탐색
제공된 `grid` 배열을 기반으로 `(0,1)`에서 `(20,19)`까지의 유효한 경로를 BFS 또는 DFS로 탐색합니다. 이때 이동은 상하좌우로만 가능하며, 값이 `0`인 칸만 이동할 수 있습니다.

### 2) moveHistory 생성
각 `(x, y)` 좌표를 `index = x * 21 + y`로 변환하여 `moveHistory` 리스트를 구성합니다.

### 3) 플래그 복호화
`moveHistory`를 기반으로 Java의 `printFlag()` 로직을 Python으로 재구현합니다. `nextProbablePrime`은 `Crypto.Util.number.isPrime`를 사용해 직접 구현합니다.

```python
from Crypto.Util.number import isPrime

def next_probable_prime(n):
    n += 1
    while not isPrime(n):
        n += 1
    return n

# 정답 경로로부터 계산된 moveHistory (BFS로 도출)
move_history = [
    1, 22, 43, 64, 85, 106, 127, 148, 149, 150, 151, 152, 131, 110, 89, 68, 67, 66, 45, 24, 25, 26, 27, 28,
    49, 70, 71, 72, 51, 30, 31, 32, 33, 34, 55, 76, 77, 78, 79, 80, 81, 82, 103, 124, 123, 122, 143, 164,
    163, 162, 161, 160, 139, 118, 117, 116, 115, 114, 135, 156, 155, 154, 175, 196, 195, 194, 193, 192,
    213, 234, 233, 232, 253, 274, 275, 276, 277, 278, 299, 320, 321, 322, 301, 280, 281, 282, 261, 240,
    219, 198, 199, 200, 221, 242, 243, 244, 245, 246, 247, 248, 227, 206, 207, 208, 229, 250, 271, 292,
    313, 334, 333, 332, 331, 330, 309, 288, 287, 286, 307, 328, 349, 370, 369, 368, 389, 410, 411, 412,
    413, 414, 393, 372, 373, 374, 375, 376, 397, 418, 439
]

# 플래그 생성 로직
result = 1
for i, val in enumerate(move_history):
    prime = next_probable_prime(val)
    result *= pow(prime, i + 1)

flag = result.to_bytes((result.bit_length() + 7) // 8, 'big').decode('utf-8', errors='ignore')
print(flag)
```

실행 결과:
```
buckeye{am4z1ng_j0b_y0u_b1g_j4va_h4ck3r}
```

---

## 🚩 Flag

```
buckeye{am4z1ng_j0b_y0u_b1g_j4va_h4ck3r}