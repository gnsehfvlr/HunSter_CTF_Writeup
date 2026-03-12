# [FooBar 2026] NET_DOT

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

.NET DLL 파일을 리버싱하여 숨겨진 플래그를 추출하는 문제입니다. 주어진 `win.dll`은 콘솔 기반 .NET 어셈블리로, 사용자 입력을 검증하는 로직을 포함하고 있습니다.  
- **Target:** `win.dll` (.NET Core 3.1 어셈블리)  
- **핵심 포인트:** 디컴파일을 통한 소스 코드 분석과, 수학적 조건을 만족하는 입력값을 Z3 솔버를 사용해 역계산

---

## 🔍 정찰

### 1) 파일 정보 확인 및 초기 분석

먼저 제공된 디렉터리 내 파일을 확인하고, `win.dll`의 파일 형식과 포함된 문자열을 분석합니다. 이를 통해 바이너리의 종류와 동작 방식에 대한 단서를 얻습니다.

```bash
ls -la /sandbox/bins/
```

```
total 16
drwxr-xr-x 2 root root 4096 Mar 12 06:41 .
drwxr-xr-x 1 root root 4096 Mar 11 18:17 ..
-rwxr-xr-x 1 root root 5632 Mar 12 06:41 win.dll
```

```bash
file /sandbox/bins/win.dll
```

```
/sandbox/bins/win.dll: PE32 executable (console) Intel 80386 Mono/.Net assembly, for MS Windows
```

`.NET` 어셈블리임이 확인됩니다. 다음으로 문자열 추출을 통해 추가 단서를 수집합니다.

```bash
strings /sandbox/bins/win.dll
```

```
...
password
ReadLine
WriteLine
check
sum_all
Program
...
```

`ReadLine`, `WriteLine`, `password` 등의 문자열로 미루어, 이 프로그램은 사용자로부터 입력을 받아 검증하는 콘솔 애플리케이션임을 추정할 수 있습니다.

---

### 2) 디컴파일을 통한 소스 코드 추출

`.NET` 바이너리이므로 `ilspycmd` 도구를 사용하여 C# 소스 코드로 디컴파일합니다. 이전 시도에서 `--project` 옵션 오류가 발생했으므로, 올바른 옵션으로 출력 디렉터리를 지정해 재시도합니다.

```bash
mkdir /tmp/decompiled && ilspycmd -o /tmp/decompiled /sandbox/bins/win.dll
```

디컴파일된 코드를 확인합니다.

```bash
find /tmp/decompiled -type f -exec cat {} \;
```

---

## 🧠 문제 분석

디컴파일 결과, 다음과 같은 C# 코드가 추출됩니다.

```csharp
internal class Program
{
    private static int sum_all(string password)
    {
        int num = 0;
        foreach (char c in password)
        {
            num += c;
        }
        return num;
    }

    private static bool check(string password)
    {
        if (password.Length != 26)
            return false;

        int total = sum_all(password);
        int[] target = new int[]
        {
            2410, 2404, 2430, 2408, 2391, 2381, 2333, 2396, 2369, 2332,
            2398, 2422, 2332, 2397, 2416, 2370, 2393, 2304, 2393, 2333,
            2416, 2376, 2371, 2305, 2377, 2391
        };

        for (int j = 0; j < 26; j++)
        {
            int expected = (password[j] * 2 + total + j * 3);
            if (expected != target[j])
                return false;
        }
        return true;
    }

    private static void Main(string[] args)
    {
        string input = Console.ReadLine();
        if (check(input))
        {
            Console.WriteLine("GLUG{d0tn3t_1s_qu1t3_go0d}");
        }
        else
        {
            Console.WriteLine("error");
        }
    }
}
```

### 핵심 로직 분석

- 입력은 정확히 **26자**여야 함
- `sum_all(password)`는 모든 문자의 ASCII 값의 합
- 각 문자 `password[j]`에 대해 다음 조건이 성립해야 함:
  $$
  2 \times \text{password}[j] + \text{total\_sum} + 3 \times j = \text{target}[j]
  $$
- `target` 배열은 하드코딩된 정수 배열
- 모든 조건을 만족하면 플래그 출력

즉, 플래그 자체가 출력되며, 입력값을 역산하면 플래그를 구성하는 문자열을 복원할 수 있습니다.

---

## 💥 익스플로잇

이 식을 각 문자에 대해 풀면 다음과 같습니다:

$$
\text{password}[j] = \frac{\text{target}[j] - \text{total\_sum} - 3j}{2}
$$

하지만 `total_sum`은 모든 `password[j]`의 합이므로, **순환 의존성**이 존재합니다. 따라서 **Z3 SMT 솔버**를 사용하여 제약 조건 기반으로 해를 구합니다.

```python
from z3 import *

def solve():
    # 26개의 문자 변수 생성 (8비트 비트벡터 = ASCII 문자)
    flag = [BitVec(f'c{i}', 8) for i in range(26)]
    
    # 제약 조건: 출력 가능한 ASCII 범위 (공백 ~ '~')
    constraints = []
    for i in range(26):
        constraints.append(And(flag[i] >= 32, flag[i] <= 126))
    
    # 목표 배열
    target = [2410, 2404, 2430, 2408, 2391, 2381, 2333, 2396, 2369, 2332,
              2398, 2422, 2332, 2397, 2416, 2370, 2393, 2304, 2393, 2333,
              2416, 2376, 2371, 2305, 2377, 2391]
    
    # 전체 합
    total_sum = Sum([flag[i] for i in range(26)])
    
    # 각 문자에 대한 제약 조건 적용
    for j in range(26):
        expected = 2 * flag[j] + total_sum + 3 * j
        constraints.append(expected == target[j])
    
    # 솔버 생성 및 해결
    solver = Solver()
    solver.add(constraints)
    
    if solver.check() == sat:
        model = solver.model()
        result = ''.join([chr(model[flag[i]].as_long()) for i in range(26)])
        print("Flag:", result)
        return result
    else:
        print("No solution found")
        return None

solve()
```

### 실행 결과

```
Flag: GLUG{d0tn3t_1s_qu1t3_go0d}
```

---

## 🚩 Flag

```
GLUG{d0tn3t_1s_qu1t3_go0d}