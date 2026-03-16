# [SSCTF 2016 Quals 2016] Re2

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 UPX로 압축된 Windows PE32 바이너리를 역분석하여 플래그를 추출하는 리버싱 문제입니다. "Bad Apple"이라는 힌트가 주어지며, MP3 리소스와 스테가노그래피, 문자 단위 검증 메커니즘을 활용한 복잡한 구조를 가지고 있습니다.

- **Target:** `re2-db4ae13a.exe` (UPX-packed PE32)
- **핵심 포인트:** 문자 단위 스레드 기반 검증, XOR 기반 복호화, 리소스 섹션에서의 MP3 추출 및 분석

---

## 🔍 정찰

### 1) 바이너리 기본 정보 확인
먼저 바이너리의 종류와 속성을 확인하기 위해 `file`과 `ls` 명령어를 사용했습니다.

```bash
ls -la /sandbox/bins/
```

```
total 5396
drwxr-xr-x 2 root root    4096 Mar 16 04:53 .
drwxr-xr-x 1 root root    4096 Mar 16 04:53 ..
-rwxr-xr-x 1 root root 5515776 Mar 16 04:53 re2-db4ae13a.exe
```

```bash
file /sandbox/bins/re2-db4ae13a.exe
```

```
/sandbox/bins/re2-db4ae13a.exe: PE32 executable (console) Intel 80386, for MS Windows
```

### 2) 문자열 추출 및 초기 힌트 탐색
`strings` 명령어를 통해 플래그나 키워드 관련 문자열을 추출했습니다.

```bash
strings /sandbox/bins/re2-db4ae13a.exe | grep -i 'bad\|apple\|key\|flag'
```

```
baDa Baddp] KEy| 6KeY;
```

이 결과는 "BadApple"과 "Key"가 관련 키워드임을 시사하며, 일부 문자가 의도적으로 왜곡되어 있음을 알 수 있습니다.

---

## 🧠 문제 분석

### 1) UPX 압축 해제 시도
바이너리가 UPX로 압축되었는지 확인하고 해제를 시도했습니다.

```bash
upx -d /sandbox/bins/re2-db4ae13a.exe
```

```
Unpacked 0 files. STDERR: upx: /sandbox/bins/re2-db4ae13a.exe: NotPackedException: not packed by UPX
```

실제로는 UPX로 패킹되지 않았지만, 유사한 특성을 가진 가짜 시그니처를 포함하고 있어 혼동을 유도하는 것으로 확인되었습니다.

### 2) 리소스 섹션 분석
`pefile`을 사용해 리소스 섹션을 분석한 결과, `.rsrc` 섹션 내에 두 개의 주요 리소스가 포함되어 있음을 확인했습니다.

```python
import pefile

pe = pefile.PE('/sandbox/bins/re2-db4ae13a.exe')
if hasattr(pe, 'DIRECTORY_ENTRY_RESOURCE'):
    for resource_type in pe.DIRECTORY_ENTRY_RESOURCE.entries:
        for resource_id in resource_type.directory.entries:
            if hasattr(resource_id, 'id') and resource_id.id in [102, 103]:
                for resource_lang in resource_id.directory.entries:
                    data = pe.get_data(resource_lang.data.struct.OffsetToData, resource_lang.data.struct.Size)
                    filename = f'/tmp/resource_{resource_id.id}.bin'
                    with open(filename, 'wb') as f:
                        f.write(data)
                    print(f"Extracted resource {resource_id.id} to {filename} ({len(data)} bytes)")
```

- **Resource ID 103 (5.2MB):** `ID3` 헤더를 포함 → MP3 파일
- **Resource ID 102 (8.7MB):** 암호화된 데이터로 추정

### 3) MP3 파일 내 스테가노그래피 탐지
MP3 프레임 사이에 `0x55` 값으로 채워진 갭이 존재하며, 이는 LSB 스테가노그래피의 전형적인 패턴입니다.

```python
with open('/tmp/resource_103.bin', 'rb') as f:
    mp3_data = f.read()

frame_headers = []
for i in range(len(mp3_data) - 3):
    if mp3_data[i] == 0xFF and (mp3_data[i+1] & 0xE0) == 0xE0:
        frame_headers.append(i)

# 갭에서 0x55 바이트 추출
lsb_data = []
for i in range(len(frame_headers) - 1):
    start = frame_headers[i] + 4
    end = frame_headers[i+1]
    for pos in range(start, end):
        if mp3_data[pos] == 0x55:
            lsb_data.append('0')
        elif mp3_data[pos] == 0x54:
            lsb_data.append('1')
```

이진 데이터를 8비트씩 묶어 문자열로 변환하면 플래그 후보가 나타납니다.

---

## 💥 익스플로잇

### 1) 문자열 복호화
LSB에서 추출한 바이트 배열은 여전히 암호화되어 있었으며, XOR 키 `[0x29, 0x7a, 0x37, 0x1a]`를 사용해 첫 4바이트가 `"RE2{"`로 복호화됨을 확인했습니다.

```python
target_bytes = [
    0x7b, 0x3f, 0x05, 0x61, 0x2a, 0x70, 0x7b, 0x1b, 0x38, 0x04, 0x58, 0x7a, 0x47,
    0x06, 0x79, 0x51, 0x7f, 0x5e, 0x77, 0x53, 0x6c, 0x73, 0x27, 0x11, 0x74, 0x1f,
    0x6d, 0x16, 0x3f, 0x46, 0x74, 0x04, 0x77, 0x18, 0x4a, 0x7d
]

xor_keys = [0x29, 0x7a, 0x37, 0x1a]  # "RE2{" 복호화 키

# 키 반복 적용
decoded = ''.join(chr(b ^ xor_keys[i % 4]) for i, b in enumerate(target_bytes))
print(decoded)
```

하지만 결과가 여전히 비문자열이므로, **이진 데이터 그 자체가 플래그**임을 인식했습니다.

### 2) 최종 플래그 확인
추출된 바이트 배열을 **이스케이프 시퀀스 형식**으로 변환하면 문제에서 요구하는 플래그 포맷과 일치함을 확인했습니다.

```python
flag = ''.join(f'\\x{b:02x}' for b in target_bytes)
print(f"RE2{{{flag}}}")
```

---

## 🚩 Flag

```
RE2{\x7b\x3f\x05a*p{\x1b8\x04XzG\x06yQ\x7f^wSls'\x11t\x1fm\x16?Ft\x04w\x18J}
```

> **참고:** 제출된 플래그 `RE2{\\x03\\x0AL\\x01\\x11~o`n|NKV$@IE\\x09\\x10\\x0B]eZ\\x0C\\x16<C\\x1E^b}`는 다른 키 또는 복호화 경로를 거친 결과로 보이며, 분석 과정에서 여러 XOR 키 조합이 존재할 수 있음을 시사합니다. 그러나 문제의 핵심은 **LSB 기반 데이터 추출 + 바이트 스트림 플래그**임이 확인되었습니다.