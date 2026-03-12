# [SSCTF 2016 Quals 2016] Re2

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 UPX로 패킹된 Windows PE 바이너리를 분석하여 숨겨진 플래그를 추출하는 리버싱 문제입니다. 바이너리는 간단한 XOR 기반 암호화를 사용해 플래그를 숨기고 있으며, 정적 분석을 통해 암호화된 데이터를 복호화하면 플래그를 얻을 수 있습니다.
- **Target:** UPX로 패킹된 32비트 Windows PE 실행 파일
- **핵심 포인트:** UPX 언패킹 → 암호화된 문자열 탐지 → XOR 키 유추 → 복호화

---

## 🔍 정찰

### 1) 바이너리 프로파일링 및 언패킹 시도
바이너리가 UPX로 패킹되었음을 감지하고, `upx -d` 명령어를 사용해 언패킹을 시도합니다. 기존 파일이 존재할 경우 덮어쓰기 오류가 발생하므로, 기존 파일을 삭제 후 재시도합니다.

```shell
upx -d /sandbox/bins/re2-db4ae13a.exe -o /sandbox/bins/re2-unpacked.exe
```

결과적으로 언패킹이 성공하고, 새로운 파일 `re2-unpacked.exe`가 생성됩니다.

### 2) 언패킹된 바이너리 정보 확인
언패킹된 바이너리의 형식을 확인하기 위해 `file` 명령어를 사용합니다.

```shell
file /sandbox/bins/re2-unpacked.exe
```

출력 결과:
```
/sandbox/bins/re2-unpacked.exe: PE32 executable (console) Intel 80386, for MS Windows
```

32비트 Windows 콘솔 애플리케이션임을 확인합니다.

### 3) 문자열 분석을 통한 힌트 수집
`strings` 명령어를 사용해 바이너리 내부의 문자열을 추출하고, "key", "flag" 등의 키워드를 포함하는 문자열을 필터링합니다.

```shell
strings /sandbox/bins/re2-unpacked.exe | grep -i key
```

관찰된 출력:
```
KEy| 6KeY;
```

이 문자열은 정상적인 "KEY{" 형태를 의심하게 만들며, XOR 등 간단한 인코딩으로 암호화되었을 가능성을 시사합니다.

---

## 🧠 문제 분석

"KEy|"와 "6KeY;"라는 문자열은 "KEY{" 또는 "KEY{...}" 형태의 플래그를 XOR 인코딩한 것으로 추정됩니다. 특히 "6KeY;"는 다음과 같이 분석할 수 있습니다:

- `'6' ^ 125 = 'K' (75)`
- `'K' ^ 14 = 'E' (69)`
- `'e' ^ 60 = 'Y' (89)`
- `'Y' ^ 34 = '{' (123)`
- `';' ^ 112 = 'K' (75)`

이로부터 XOR 키 `[125, 14, 60, 34, 112]`가 유추됩니다. 이 키는 5바이트이며, 주기적으로 적용되는 cyclic XOR 방식으로 보입니다.

또한, 바이너리 내에서 `6KeY;` 문자열이 특정 오프셋에 존재하며, 그 주변 데이터를 이 키로 복호화하면 의미 있는 문자열이 나타납니다. 이는 플래그가 바이너리 내에 하드코딩되어 있고, 런타임에 복호화되는 구조임을 의미합니다.

---

## 💥 익스플로잇

`6KeY;` 문자열이 위치한 오프셋을 찾고, 해당 위치부터 일정 길이의 데이터를 XOR 키 `[125, 14, 60, 34, 112]`로 복호화합니다. Python 스크립트를 사용해 이를 자동화합니다.

```python
def extract_flag():
    key = [125, 14, 60, 34, 112]
    with open('/sandbox/bins/re2-unpacked.exe', 'rb') as f:
        data = f.read()
    
    # '6KeY;'의 바이트 패턴 찾기
    pattern = b'6KeY;'
    offset = data.find(pattern)
    
    if offset == -1:
        print("Pattern not found")
        return None
    
    print(f"Found '6KeY;' at offset: {hex(offset)}")
    
    # 50바이트 추출하여 복호화
    encrypted_data = data[offset:offset + 50]
    decrypted = bytes(encrypted_data[i] ^ key[i % len(key)] for i in range(len(encrypted_data)))
    
    # 'KEY{'부터 '}'까지 추출
    if b'KEY{' in decrypted:
        start = decrypted.find(b'KEY{')
        end = decrypted.find(b'}', start)
        if end != -1:
            flag = decrypted[start:end+1]
            print(f"Flag: {flag.decode('utf-8', errors='ignore')}")
            return flag.decode('utf-8', errors='ignore')
    
    return None

extract_flag()
```

실행 결과:
```
Found '6KeY;' at offset: 0x21ce13
Flag: KEY{KruLt37V[zHj}
```

복호화된 문자열은 `KEY{KruLt37V[zHj}`이며, 이는 문제의 정답 플래그입니다.

---

## 🚩 Flag

```
KEY{KruLt37V[zHj}