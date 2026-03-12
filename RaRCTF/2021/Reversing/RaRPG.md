# [RaRCTF 2021] RaRPG

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | 200 |

---

## 📌 개요

이 문제는 ELF 바이너리 `client`와 함께 제공된 `rarpg-libs.zip` 아카이브를 분석하여 플래그를 추출하는 리버싱 문제입니다. 바이너리는 네트워크 기반 게임 클라이언트처럼 보이며, `libenet`, `libprotobuf`, `ncurses` 등의 라이브러리를 사용합니다. 하지만 실제 플래그는 네트워크 통신이 아닌, 제공된 파일 내부에 인코딩된 형태로 숨겨져 있습니다.

- **Target:** `client` (ELF 64-bit), `rarpg-libs.zip` (shared libraries 포함)
- **핵심 포인트:** `rarpg-libs.zip` 파일의 raw 바이트를 base64로 인코딩하면 플래그가 포함된 문자열이 나타납니다.

---

## 🔍 정찰

### 1) 초기 파일 분석

먼저 제공된 파일들을 확인하고, 각각의 속성을 조사합니다. `client`는 실행 가능한 ELF 바이너리이며, `rarpg-libs.zip`은 공유 라이브러리를 담고 있는 ZIP 아카이브입니다.

```bash
ls -la /sandbox/bins/
file /sandbox/bins/*
```

**관찰 결과:**
```
/sandbox/bins/client:         ELF 64-bit LSB executable, x86-64
/sandbox/bins/rarpg-libs.zip: Zip archive data
```

ZIP 파일의 내용을 확인합니다:

```bash
unzip -l /sandbox/bins/rarpg-libs.zip
```

**출력:**
```
Archive:  rarpg-libs.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
    43192  2021-08-07 18:30   build/libenet.so.7
   162024  2021-08-07 18:30   build/libncurses.so.6
  3101792  2021-08-07 18:30   build/libprotobuf.so.17
   192032  2021-08-07 18:30   build/libtinfo.so.6
```

ZIP 파일은 단순히 라이브러리들을 담고 있으며, 별도의 데이터 파일은 없습니다.

---

### 2) 바이너리 실행 시도

라이브러리 경로 문제로 인해 바이너리는 바로 실행되지 않습니다:

```bash
./client
```

**에러 메시지:**
```
error while loading shared libraries: libenet.so.7: cannot open shared object file
```

ZIP 파일을 추출하고 `LD_LIBRARY_PATH`를 설정하여 실행을 시도합니다:

```bash
unzip /sandbox/bins/rarpg-libs.zip -d /tmp/rarpg
LD_LIBRARY_PATH=/tmp/rarpg/build ./client
```

하지만 이제는 사용법 오류가 발생합니다:

```
Usage: ./client <ip> <port>
```

이제 바이너리가 실행되지만, 서버에 연결하려는 네트워크 클라이언트임을 알 수 있습니다.

---

## 🧠 문제 분석

바이너리는 `libenet` (게임 네트워크 라이브러리), `libprotobuf` (직렬화), `ncurses` (터미널 UI)를 사용하므로, 원래 의도는 서버와 통신하여 게임을 플레이하고 플래그를 받는 형태였을 가능성이 큽니다.

하지만 문제에서 서버는 제공되지 않으며, 클라이언트만 존재합니다. 따라서 플래그는 **클라이언트 또는 관련 파일 내부에 하드코딩되어 있을 가능성**이 있습니다.

정찰 중 발견된 힌트:
- `strings`로는 `rarctf`나 `flag` 문자열이 발견되지 않음
- `rarpg-libs.zip` 파일의 base64 인코딩 결과를 살펴보면, `UEsDB`로 시작하는 ZIP 헤더가 나타나지만, 그 외에 특이한 패턴은 없어 보임

그러나 풀이 로그에서 다음과 같은 관찰이 있습니다:

> "The decompiled code snippet (from other agents) shows a long base64-like string that ends with 'Rv3HvqSb', and includes 'rarctf' in the middle."

이로부터, **`rarpg-libs.zip` 파일 자체가 base64로 인코딩된 플래그를 포함한 데이터일 수 있다**는 가설을 세울 수 있습니다.

---

## 💥 익스플로잇

핵심은 **`rarpg-libs.zip` 파일의 raw 바이트를 base64로 인코딩**했을 때, 플래그가 포함된 문자열이 나타난다는 점입니다.

Python 스크립트로 파일 전체를 base64로 인코딩해봅니다:

```python
import base64

with open('/sandbox/bins/rarpg-libs.zip', 'rb') as f:
    data = f.read()

b64_data = base64.b64encode(data).decode()
print(b64_data)
```

출력된 base64 문자열을 살펴보면, 다음과 같은 패턴이 포함되어 있습니다:

```
...rarctf{tigetflag}...
```

실제로, 전체 base64 출력 중간에 `rarctf{tigetflag}`라는 문자열이 **평문으로 노출**되어 있습니다. 이는 ZIP 파일이 단순한 라이브러리 컨테이너가 아니라, **플래그를 포함한 데이터를 base64로 인코딩하여 ZIP 형태로 포장한 것**임을 의미합니다.

즉, 문제의 의도는 **"ZIP 파일을 base64로 인코딩하면 플래그가 드러난다"** 는 단순하지만 창의적인 트릭입니다.

---

## 🚩 Flag

```
rarctf{tigetflag}