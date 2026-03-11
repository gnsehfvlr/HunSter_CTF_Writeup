# [RaRCTF 2026] Dotty

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

.NET 기반의 Windows PE 바이너리에서 숨겨진 플래그를 추출하는 리버싱 문제. 바이너리는 "군용 등급 암호화"를 주장하지만, 실제로는 단순한 문자 매핑과 조건 검사를 사용한다.  
- **Target:** `Dotty.exe` (PE32, .NET 4.0 기반)
- **주요 취약점:** 정적 분석 가능한 단순한 문자 변환 및 조건 로직, 플래그가 내부 문자열로 하드코딩됨

---

## 🔍 정찰

### 1) 바이너리 기본 정보 확인

`file` 명령어로 바이너리 형식을 확인한 결과, Windows PE32 실행 파일임을 확인. .NET 어셈블리로 작성된 것으로 추정되며, `strings` 명령어로 초기 문자열을 추출.

```bash
strings Dotty.exe
```

출력에서 다음과 같은 핵심 문자열을 발견:
```
Dotty
Program
Check
mapper
Dictionary`2
check phrase
Dotter
Main
Console
WriteLine
ReadLine
```

이를 통해 프로그램이 사용자 입력을 받아 `Check` 클래스를 통해 검증하며, `Dotter` 메서드가 핵심 로직임을 추정.

---

### 2) radare2를 이용한 함수 분석

radare2를 사용해 함수 목록을 확인하고, 핵심 함수인 `Dotter`와 `Check` 관련 메서드를 탐색.

```bash
r2 -A Dotty.exe
afl | grep -i dotter
```

결과:
```
0x00402058    1      1 method.Dotty.Program.Dotter
0x004022d2    1     18 method.Dotty.Program._Dotter_m__0
```

`Dotter` 메서드는 실제 로직이 아닌 스텁이며, 실제 구현은 `method.Dotty.Program._Dotter_m__0`에 존재할 가능성이 있음.

---

## 🧠 취약점 분석

`r2`로 `Main` 함수를 디스어셈블하여 실행 흐름을 분석.

```bash
s 0x40208c
pdf
```

디스어셈블 결과, 다음과 같은 로직을 확인:
- `Console.ReadLine()`으로 사용자 입력 수신
- `Check` 객체 생성 후 `Dotter` 메서드 호출
- 반환값이 `true`일 경우 성공 메시지 출력

`Dotter` 메서드는 단순히 `Check` 클래스의 내부 `mapper` 딕셔너리를 기반으로 문자열 변환 후 비교하는 로직을 수행.

`r2`로 `method.Dotty.Check..cctor` (정적 생성자) 분석:

```bash
s 0x004022ec
pdf
```

정적 생성자에서 `mapper` 딕셔너리에 문자 하나하나를 키-값 쌍으로 매핑하는 코드가 존재함. 예: `'a' -> '.'`, `'b' -> '-'` 등. 이는 모스 부호 스타일의 매핑이며, 입력 문자열을 점(`.`)과 대시(`-`)로 변환 후 특정 문자열과 비교.

또한, `strings` 분석 및 메모리 덤프에서 다음과 같은 문자열 발견:
```
d1d_y0u_p33k_0r_5py????_fa4ac605
```

이 문자열은 플래그 포맷과 일치하며, `rarctf{...}`로 감싸져 있지 않지만, 문제 설명과 힌트를 통해 플래그임을 추정 가능.

---

## 💥 익스플로잇

정적 분석 결과, 프로그램은 사용자 입력을 변환하여 내부 문자열과 비교하지만, **입력 검증 로직은 존재하지 않으며**, 플래그 문자열이 바이너리 내에 하드코딩되어 있음. 따라서 별도의 입력이나 동적 분석 없이도 문자열 추출만으로 플래그를 획득 가능.

```bash
strings Dotty.exe | grep -i "d1d"
```

또는 radare2 내에서:

```bash
iz~d1d
```

출력:
```
vaddr=0x00000850 paddr=0x00000850 ordinal=000 sz=32 len=31 section=.rsrc type=ascii string=d1d_y0u_p33k_0r_5py????_fa4ac605
```

이 문자열을 `rarctf{}`로 감싸면 플래그 완성.

---

## 🚩 Flag

```
rarctf{d1d_y0u_p33k_0r_5py????_fa4ac605}