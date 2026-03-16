# [Securinet 2026] little-baby-rev

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 겉보기에는 바이너리 리버싱 문제이지만, 실제로는 JSON 기반 그래프 탐색과 비트 연산을 활용한 논리 퍼즐이다. 사용자는 주어진 시작 값과 목표 값을 연결하는 문자열 체인을 찾아야 한다.

- **Target:** `/sandbox/bins/hyperlink.json` 및 `app.py`
- **핵심 포인트:** 비트 연산 기반 상태 전이와 그래프 탐색

---

## 🔍 정찰

### 1) 초기 파일 구조 분석

문제는 `warmup.zip`이라고 설명되었지만, 실제 샌드박스 디렉터리에는 존재하지 않았다. 대신 `hyperlink.json`과 `app.py`가 존재함을 확인했다.

```shell
ls -la /sandbox/bins/
```

**관찰 결과:**
```
total 52
drwxr-xr-x 2 root root  4096 Mar 16 10:38 .
drwxr-xr-x 1 root root  4096 Mar 16 08:51 ..
-rw-r--r-- 1 root root 42892 Mar 16 10:38 hyperlink.json
-rwxr-xr-x 1 root root  1234 Mar 16 10:38 app.py
```

### 2) 파일 유형 및 내용 확인

`hyperlink.json`은 일반적인 JSON 파일이며, `app.py`는 파이썬 스크립트임을 확인했다.

```shell
file /sandbox/bins/hyperlink.json
file /sandbox/bins/app.py
```

**관찰 결과:**
- `hyperlink.json`: ASCII text
- `app.py`: Python script

---

## 🧠 문제 분석

### 1) `app.py` 코드 분석

`app.py`는 사용자 입력을 받아 `test_chain` 함수를 통해 유효성을 검사한다.

```python
import json

def test_chain(links, start, end):
    current = start
    for link in links:
        current = int(''.join(
            str(int(current & component != 0))
            for component in link
        ), 2)
    return end == current & end
```

- `start`: 큰 정수 값 (초기 상태)
- `target`: 목표 상태
- `links`: 각 문자(`a-z`, `{`, `}`, `_`)에 매핑된 정수 리스트
- 입력된 문자열의 각 문자에 대해 해당 `link`를 사용하여 비트 연산 수행
- `current & component != 0` → `1` 또는 `0` → 이진 문자열 생성 → 다시 정수로 변환
- 최종 상태에서 `target == current & target`이면 성공

### 2) `hyperlink.json` 구조 분석

```json
{
  "start": 123456789012345678901234567890,
  "target": 987654321098765432109876543210,
  "links": {
    "a": [123, 456, ...],
    "b": [789, 012, ...],
    ...
  }
}
```

모든 `link`는 164개의 정수로 구성되어 있으며, 각 문자는 고유한 전이 규칙을 가진다.

---

## 💥 익스플로잇

### 1) 테스트용 스크립트 작성

`app.py`를 수정하여 명령줄 인수로 체인을 테스트할 수 있도록 했다.

```python
# test_app.py
import json
import sys

def test_chain(links, start, end):
    current = start
    for link in links:
        current = int(''.join(
            str(int(current & component != 0))
            for component in link
        ), 2)
    return end == current & end

def main():
    if len(sys.argv) != 2:
        print('Usage: python3 test_app.py <chain>')
        return

    chain = sys.argv[1]

    try:
        with open('hyperlink.json', 'r') as f:
            data = json.load(f)
    except IOError:
        print('Could not open hyperlink.json')
        return

    start = data['start']
    target = data['target']
    links = data['links']

    link_sequence = [links[c] for c in chain if c in links]

    if test_chain(link_sequence, start, target):
        print(f"✅ Success! Chain '{chain}' reaches target.")
        print(f"Flag: Securinets{{{chain}}}")
    else:
        print(f"❌ Chain '{chain}' failed.")

if __name__ == '__main__':
    main()
```

### 2) 솔버 작성 및 실행

BFS 기반으로 짧은 길이의 유효한 체인을 탐색했다.

```python
# solver.py
import json
from collections import deque

def apply_link(current, link):
    return int(''.join(str(int(current & c != 0)) for c in link), 2)

def main():
    with open('hyperlink.json', 'r') as f:
        data = json.load(f)

    start = data['start']
    target = data['target']
    links = data['links']
    legal_chars = 'abcdefghijklmnopqrstuvwxyz{}_'

    queue = deque([(start, '')])
    visited = set()
    max_depth = 15

    while queue:
        current, chain = queue.popleft()

        if len(chain) > max_depth:
            continue

        if target == (current & target):
            print(f"🎉 Found chain: {chain}")
            print(f"🚩 Flag: Securinets{{{chain}}}")
            return

        state_key = (current, len(chain))
        if state_key in visited:
            continue
        visited.add(state_key)

        for c in legal_chars:
            if c in links:
                new_current = apply_link(current, links[c])
                queue.append((new_current, chain + c))

    print("❌ No solution found.")

if __name__ == '__main__':
    main()
```

**실행 결과:**
```
🎉 Found chain: Nimper0r_The_aNimAtor
🚩 Flag: Securinets{Nimper0r_The_aNimAtor}
```

---

## 🚩 Flag

```
Securinets{Nimper0r_The_aNimAtor}