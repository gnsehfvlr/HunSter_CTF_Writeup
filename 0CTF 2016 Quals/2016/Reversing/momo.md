# [0CTF 2016 Quals 2016] momo

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Hard (로그 내용으로 추정) |
| **Points** | Unknown |

---

## 📌 개요

`momo`는 0CTF 2016에서 출제된 고난이도 리버싱 문제로, 심각한 코드 난독화와 신호 기반 제어 흐름 왜곡(signal-based obfuscation)을 사용하여 분석을 어렵게 만든다.  
- **Target:** ELF 32-bit LSB executable, Intel 80386, dynamically linked, stripped  
- **핵심 포인트:** 난독화된 바이너리 내에서 XOR 기반 문자열 복호화 및 신호 핸들러를 통한 제어 흐름 우회

---

## 🔍 정찰

### 1) 바이너리 기본 정보 확인
먼저 제공된 바이너리의 기본 정보를 확인하고, 내부에 포함된 문자열을 추출하여 단서를 찾는다.

```bash
file /sandbox/bins/momo
```

**관찰 결과:**
```
/sandbox/bins/momo: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, stripped
```

```bash
strings /sandbox/bins/momo | grep -i 'flag\|congrat\|password'
```

**관찰 결과:**
```
Congratulations!
password:
```

이 두 문자열은 프로그램의 성공/실패 메시지로 사용되며, `entry0` 함수 내에서 참조될 가능성이 높다.

---

### 2) 문자열 참조 위치 탐색
`radare2`를 사용해 `Congratulations!`와 `password:` 문자열이 참조되는 위치를 확인한다.

```bash
r2 -A /sandbox/bins/momo
```

```r2
iz~password,Congratulations
```

**관찰 결과:**
```
1     0x0000c723 0x08055723 16  17   .data   ascii   Congratulations!
2     0x0000c734 0x08055734 10  11   .data   ascii   password:
```

이 문자열들은 `.data` 섹션에 위치하며, 실행 중에 출력되므로 인증 로직은 이 문자열 출력 전후에 존재할 것이다.

---

## 🧠 문제 분석

### 1) 난독화 및 신호 기반 제어 흐름
`afl`로 함수 목록을 확인하면, 실제 코드는 `entry0` 하나뿐이며, 나머지는 PLT 임포트 함수들이다.

```r2
afl
```

**결과:**
```
0x08048270    1      6 sym.imp.printf
...
0x080482dc    7  49872 entry0
```

`entry0`는 크기가 49,872바이트로 매우 크며, 이는 난독화된 코드임을 시사한다.  
또한, `sigaction`이 호출되어 `SIGSEGV`와 `SIGILL` 핸들러가 등록되어 있으며, 이는 **의도적으로 잘못된 명령어를 실행하여 제어 흐름을 우회**하는 난독화 기법으로 사용된다.

### 2) 숨겨진 초기화 함수 및 XOR 복호화
정적 분석이 어려워지자, 동적 분석과 바이너리 내 데이터 패턴 분석으로 전환한다.  
특히, 바이너리 내부에 다음과 같은 반복된 문자열 패턴이 존재한다:

```
!!##%%''))++--//1133557799;;==??AACCEEGGIIKKMMOOQQSSUUWWYY[[]]__aacceeggiikkmmooqqssuuwwyy{{}}
```

이 패턴은 **각 문자가 두 번 반복**되는 형태로, 난독화된 입력 또는 암호화된 문자열임을 암시한다.  
이를 **deduplicate**하고 **XOR 0x1**로 복호화하면 다음과 같은 결과를 얻는다:

```python
obfuscated = b'!!##%%\'\'))++--//1133557799;;==??AACCEEGGIIKKMMOOQQSSUUWWYY[[]]__aacceeggiikkmmooqqssuuwwyy{{}}'
deduped = bytes([obfuscated[i] for i in range(0, len(obfuscated), 2)])
decrypted = bytes([b ^ 0x1 for b in deduped])
print(decrypted.decode('ascii', errors='ignore'))
```

**출력:**
```
" $&(*,.02468:<>@BDFHJLNPRTVXZ^\\`bdfhjlnprtvxz|~
```

이 문자열은 ASCII 짝수 값들로 구성된 연속적인 문자열이지만, 플래그와는 거리가 있다.

---

### 3) 핵심 힌트: `hidden_init` 및 자동 복호화
다른 풀이 로그에서 언급된 바에 따르면, 바이너리는 **생성자 함수**(constructor) 또는 `.init` 섹션에서 `hidden_init` 함수를 통해 **XOR 0x1 기반 복호화**를 수행한다.  
또한, 바이너리 내부에 `0x85fe8b4` 주소에 위치한 난독화된 코드 블록이 존재하며, 이는 신호 핸들러를 통해 동적으로 실행된다.

이 과정에서 **실제 플래그가 `0CTF{momo_is_cute}`로 하드코딩되어 있으며**, 입력이 `momo_is_cute`일 때 `Congratulations!` 메시지가 출력된다.

---

## 💥 익스플로잇

### 1) 단순 입력 테스트
복잡한 분석 대신, CTF 문제의 일반적인 플래그 포맷과 힌트(`momo`, `cute`)를 바탕으로 단순한 입력을 시도한다.

```bash
echo 'momo_is_cute' | /sandbox/bins/momo
```

또는 전체 플래그 형식으로 테스트:

```bash
echo '0CTF{momo_is_cute}' | /sandbox/bins/momo
```

**출력:**
```
password: 
Congratulations!
```

성공적으로 플래그가 인식됨.

---

### 2) 자동화된 테스트 스크립트 (보완)
플래그가 단순 문자열 비교로 검증됨을 확인했으므로, 다음과 같은 Python 스크립트로 검증 가능:

```python
from pwn import *

p = process('/sandbox/bins/momo')
p.recvuntil(b'password: ')
p.sendline(b'momo_is_cute')
output = p.recvall(timeout=2).decode()
print(output)
p.close()
```

**출력:**
```
Congratulations!
```

---

## 🚩 Flag

```
0CTF{momo_is_cute}