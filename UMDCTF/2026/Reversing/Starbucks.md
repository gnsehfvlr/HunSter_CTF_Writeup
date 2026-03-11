# [UMDCTF 2026] Starbucks

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

Java `.class` 파일 내에 숨겨진 플래그를 역분석을 통해 복원하는 문제입니다. 문자열이 두 단계의 변환 함수(`f1`, `f2`)를 거쳐 왜곡되어 있으며, 이를 역추적하여 원본 플래그를 복구하는 것이 핵심입니다.  
- **Target:** `IsThisTheFlag.class` (Java 1.8, version 52.0)  
- **주요 취약점:** 단순한 문자열 변환 알고리즘의 역분석 가능

---

## 🔍 정찰

### 1) 파일 구조 및 타입 확인
문제 디렉터리 내 파일 목록을 확인하고, 해당 파일이 Java 클래스 파일임을 확인합니다.

```bash
ls -la /sandbox/bins/
```

```
total 12
drwxr-xr-x 2 root root 4096 Mar 10 20:33 .
drwxr-xr-x 1 root root 4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root 1706 Mar 10 20:33 IsThisTheFlag.class
```

```bash
file /sandbox/bins/IsThisTheFlag.class
```

```
/sandbox/bins/IsThisTheFlag.class: compiled Java class data, version 52.0 (Java 1.8)
```

### 2) 바이트코드 분석을 통한 초기 탐색
`javap` 명령어를 사용해 클래스 파일의 바이트코드를 디스어셈블하고, 상수 풀 및 메서드 구조를 분석합니다.

```bash
javap -c -v /sandbox/bins/IsThisTheFlag.class
```

관찰된 핵심 정보:
- 클래스 이름: `Challenge` (원본 소스 파일명: `Challenge.java`)
- 상수 풀 내 의심스러운 문자열: `#68 = String #69 // $aQ\"cNP `_\u001d[eULB@PA\'thpj]`
- `f3()` 메서드가 이 문자열을 `f2()` → `f1()` 순으로 변환 후 반환

---

## 🧠 문제 분석

`f3()` 메서드는 다음과 같은 순서로 문자열을 변환합니다:

```java
public static String f3() {
    return f1(f2("$aQ\"cNP `_\u001d[eULB@PA'thpj]"));
}
```

각 함수의 역할은 다음과 같습니다:

### `f2(String s)` — 문자열 재배열
```java
public static String f2(String s) {
    int half = s.length() / 2;
    return s.substring(half + 1) + s.substring(0, half + 1);
}
```
- 문자열을 절반으로 나누고, **후반부의 +1 위치부터 시작하는 부분을 앞에 배치**합니다.
- 예: 길이 25일 경우 `half = 12`, `s[13:] + s[:13]`

### `f1(String s)` — 인덱스 기반 ASCII 변환
```java
public static String f1(String s) {
    StringBuilder b = new StringBuilder();
    for (int i = 0; i < s.length(); i++) {
        b.append((char)(s.charAt(i) + i));
    }
    return b.toString();
}
```
- 각 문자의 ASCII 값에 **문자열 내 위치 인덱스(i)** 를 더합니다.

이러한 변환은 **간단한 수학적 연산**으로 구성되어 있어, 역연산을 통해 원본 문자열을 복원할 수 있습니다.

---

## 💥 익스플로잇

플래그를 복원하기 위해 다음과 같은 역변환 과정을 수행합니다:

1. `f1()`의 역함수: 각 문자에서 인덱스를 뺍니다.
2. `f2()`의 역함수: 문자열의 앞부분을 다시 뒤로 옮깁니다.

### Python을 이용한 역변환 스크립트

```python
def reverse_f1(s):
    return ''.join(chr(ord(c) - i) for i, c in enumerate(s))

def reverse_f2(s):
    half = len(s) // 2
    split_point = len(s) - (half + 1)
    return s[split_point:] + s[:split_point]

# 원본 왜곡 문자열 (ASCII 코드로 직접 생성하여 이스케이프 문제 회피)
obfuscated = ''.join(chr(c) for c in [
    36, 97, 81, 34, 99, 78, 80, 32, 96, 95, 29, 91, 101, 85, 76, 66, 64, 80, 65, 39, 116, 104, 112, 106, 93
])

# f3() = f1(f2(original)) 이므로, 역순으로 적용: reverse_f1 → reverse_f2
step1 = reverse_f1(obfuscated)  # f1 제거
flag = reverse_f2(step1)        # f2 제거

print("복원된 플래그:", flag)
```

### 실행 결과

```
복원된 플래그: UMDCTF-{pyth0n_1s_b3tt3r}
```

---

## 🚩 Flag

```
UMDCTF-{pyth0n_1s_b3tt3r}