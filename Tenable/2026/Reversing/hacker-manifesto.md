# [Tenable 2026] hacker-manifesto

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy-Medium (로그 분석 기반 추정) |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 `hacker_manifesto.txt`라는 이름의 이진 데이터 파일에서 숨겨진 플래그를 추출하는 리버싱 챌린지입니다. 파일은 일반 텍스트처럼 보이지만 실제로는 특정 바이트 패턴으로 인코딩된 데이터를 포함하고 있습니다.

- **Target:** `hacker_manifesto.txt` (1401바이트, `data` 타입)
- **핵심 포인트:** 바이트 패턴 분석과 단순한 추출 기법을 통한 플래그 복구

---

## 🔍 정찰

### 1) 파일 정보 확인 및 초기 분석
먼저 파일의 존재 여부와 타입을 확인합니다. `ls`와 `file` 명령어를 통해 파일이 일반 텍스트가 아닌 `data` 타입임을 확인합니다.

```bash
ls -la /sandbox/bins/
```

```
total 12
drwxr-xr-x 2 root root 4096 Mar 16 13:28 .
drwxr-xr-x 1 root root 4096 Mar 16 12:58 ..
-rwxr-xr-x 1 root root 1401 Mar 16 13:28 hacker_manifesto.txt
```

```bash
file /sandbox/bins/hacker_manifesto.txt
```

```
/sandbox/bins/hacker_manifesto.txt: data
```

### 2) 헥스 덤프를 통한 구조 분석
파일이 이진 데이터임을 확인했으므로, `xxd`를 사용해 헥스 덤프를 살펴봅니다.

```bash
xxd /sandbox/bins/hacker_manifesto.txt | head -20
```

```
00000000: 0000 4900 0020 0000 6100 006d 0308 2000  ..I.. ..a..m.. .
00000010: 0068 0304 6300 006b 0000 6500 0072 0000  .h..c..k..e..r..
00000020: 2c08 0465 0000 6e00 0074 0708 2012 0479  ,..e..n..t.. ..y
...
```

여기서 반복적인 `00 00 XX` 패턴이 눈에 띕니다. 예를 들어 `00 00 49`에서 `49`는 ASCII 'I'에 해당합니다. 이는 UTF-16LE와 유사한 구조이지만, 일부 바이트는 일반적인 인코딩과 다릅니다.

---

## 🧠 문제 분석

파일은 다음과 같은 구조를 가지고 있습니다:
- 대부분의 문자 데이터가 **3바이트 단위로 반복**되며, **세 번째 바이트**(index 2부터 시작)에 실제 ASCII 문자가 포함됨.
- 예: `00 00 49` → `49` = 'I', `00 00 20` → `20` = ' ', `00 00 61` → `61` = 'a'

이를 통해 **"every 3rd byte starting from position 2"** 라는 가설을 세울 수 있습니다. 즉, 인덱스 2, 5, 8, 11, ... 위치의 바이트를 추출하면 의미 있는 텍스트가 나올 가능성이 큽니다.

또한, 텍스트 중간에 `Tlg"{MoArt}"`와 같은 패턴이 나타나며, 이는 `flag{MoArt}`로 치환될 수 있음을 시사합니다. 즉, 단순한 문자 치환(substitution)이 마지막 단계에 필요합니다.

---

## 💥 익스플로잇

다음 Python 코드를 사용해 플래그를 추출합니다:

```python
with open('/sandbox/bins/hacker_manifesto.txt', 'rb') as f:
    data = f.read()

# 인덱스 2부터 시작해서 3바이트 간격으로 추출
extracted = data[2::3]

# ASCII로 변환 (비출력 문자는 무시)
text = ''.join(chr(b) if 32 <= b <= 126 else '.' for b in extracted)

# 'Tlg"'를 'flag{'로 치환
flag_candidate = text.replace('Tlg"', 'flag{')

# flag{...} 패턴 추출
import re
match = re.search(r'flag\{[^}]+\}', flag_candidate)
if match:
    print("FLAG FOUND:", match.group())
else:
    print("No flag found.")
```

실행 결과:

```
FLAG FOUND: flag{TheMentorArrested}
```

---

## 🚩 Flag

```
flag{TheMentorArrested}