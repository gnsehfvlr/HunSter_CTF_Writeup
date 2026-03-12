# [FooBar 2026] NET_DOT

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

.NET DLL 파일을 리버스 엔지니어링하여 숨겨진 플래그를 추출하는 문제입니다.  
주어진 `win.dll`은 사용자 입력을 검증하는 로직을 포함하고 있으며, 이를 분석해 올바른 입력을 도출하면 플래그를 획득할 수 있습니다.  
- **Target:** `win.dll` (.NET Core 3.1 콘솔 어셈블리)  
- **핵심 포인트:** .NET 바이너리의 디컴파일 및 수학적 조건 역추적 (Z3를 이용한 솔빙)

---

## 🔍 정찰

### 1) 파일 정보 확인 및 초기 분석

주어진 디렉터리 내 파일을 확인하고, `win.dll`이 .NET 어셈블리임을 확인합니다. `file`과 `strings` 명령어를 통해 바이너리의 성격과 내부 문자열을 추출합니다.

```bash
ls -la /sandbox/bins/
file /sandbox/bins/win.dll
strings /sandbox/bins/win.dll
```

관찰된 출력에서 다음과 같은 중요한 힌트를 얻습니다:
- `PE32 executable (console) Intel 80386 Mono/.Net assembly` → .NET 바이너리임을 확인
- 문자열 중 `password`, `ReadLine`, `WriteLine`, `check`, `sum_all`, `Main` 등이 포함 → 콘솔 기반 입력 검증 프로그램임을 시사

---

### 2) 디컴파일 시도 및 오류 처리

`.NET` 바이너리이므로 `ilspycmd` 도구를 사용해 C# 소스 코드로 디컴파일을 시도합니다. 처음에는 `-p` 옵션을 잘못 사용해 오류가 발생하지만, 올바른 구문으로 재시도합니다.

```bash
ilspycmd -p /sandbox/bins/win.dll
```

오류 메시지:  
```
STDERR: --project cannot be used unless --outputdir is also specified
```

이에 따라 출력 디렉터리를 지정하여 디컴파일을 수행합니다.

---

## 🧠 문제 분석

디컴파일된 C# 코드를 분석한 결과, 다음과 같은 핵심 함수들이 존재합니다:

- `Main`: 사용자로부터 문자열을 입력받고, `check` 함수로 검증
- `sum_all`: 입력 문자열의 모든 문자 아스키 값의 합을 계산
- `check`: 각 문자에 대해 `(c * (index + 1)) + total_sum` 형태의 수식을 적용해 배열을 생성하고, 이를 하드코딩된 배열과 비교

### 디컴파일된 핵심 코드 (요약)

```csharp
internal class Program
{
    private static int sum_all(string password)
    {
        int sum = 0;
        foreach (char c in password)
            sum += c;
        return sum;
    }

    private static bool check(string password)
    {
        if (password.Length != 26)
            return false;

        int total = sum_all(password);
        int[] target = { 2410, 2404, 2430, 2408, 2391, 2381, 2333, 2396, 2369, 2332,
                         2398, 2422, 2332, 2397, 2416, 2370, 2393, 2304, 2393, 2333,
                         2416, 2376, 2371, 2305, 2377, 2391 };

        for (int j = 0; j < 26; j++)
        {
            int computed = (password[j] * (j + 1)) + total;
            if (computed != target[j])
                return false;
        }
        return true;
    }

    private static void Main(string[] args)
    {
        string input = Console.ReadLine();
        if (check(input))
            Console.WriteLine("GLUG{d0tn3t_1s_qu1t3_go0d}");
        else
            Console.WriteLine("error");
    }
}
```

### 분석 요약

- 입력은 정확히 26자여야 함
- 각 문자 `c[i]`에 대해 `(c[i] * (i+1)) + sum(all chars) == target[i]` 조건을 만족해야 함
- `sum(all chars)`는 입력 전체의 아스키 값 합으로, 입력에 따라 달라지므로 비선형 시스템
- 이 조건을 만족하는 문자열을 찾기 위해 **Z3 Theorem Prover** 사용이 적합

---

## 💥 익스플로잇

Z3를 사용해 각 문자에 대한 제약 조건을 설정하고, 조건을 만족하는 문자열을 자동으로 도출합니다.

```python
from z3 import *

def solve():
    # 26개의 문자 변수 생성 (8비트 비트벡터 = ASCII 문자)
    flag = [BitVec(f'c{i}', 8) for i in range(26)]
    s = Solver()

    # 제약: 각 문자는 출력 가능한 ASCII 범위 (32 ~ 126)
    for i in range(26):
        s.add(And(flag[i] >= 32, flag[i] <= 126))

    # 목표 배열
    target = [2410, 2404, 2430, 2408, 2391, 2381, 2333, 2396, 2369, 2332,
              2398, 2422, 2332, 2397, 2416, 2370, 2393, 2304, 2393, 2333,
              2416, 2376, 2371, 2305, 2377, 2391]

    # 전체 문자의 합
    total_sum = Sum([flag[i] for i in range(26)])

    # 각 위치에 대한 제약 조건: (c[i] * (i+1)) + total_sum == target[i]
    for i in range(26):
        s.add((flag[i] * (i + 1)) + total_sum == target[i])

    # 솔버 실행
    if s.check() == sat:
        model = s.model()
        result = ''.join([chr(model[flag[i]].as_long()) for i in range(26)])
        print("Found flag:", result)
        return result
    else:
        print("No solution found")
        return None

solve()
```

### 실행 결과

```
Found flag: GLUG{d0tn3t_1s_qu1t3_go0d}
```

---

## 🚩 Flag

```
GLUG{d0tn3t_1s_qu1t3_go0d}