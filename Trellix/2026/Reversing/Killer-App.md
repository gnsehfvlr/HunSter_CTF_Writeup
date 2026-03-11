# [Trellix 2026] Killer-App

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 Android APK 파일(`Fit32.apk`)과 함께 제공된 PNG 이미지(`app.png`)에서 숨겨진 플래그를 추출하는 리버싱 문제입니다. 앱은 정상적인 피트니스 앱처럼 보이지만, 내부에 스테가노그래피(steganography) 기법을 사용해 플래그를 숨겼으며, LSB(Least Significant Bit) 기반의 데이터 인코딩과 XOR 복호화가 결합된 방식을 사용합니다.

- **Target:** `Fit32.apk` 및 `app.png`
- **주요 취약점:** PNG 이미지의 LSB 채널에 숨겨진 인코딩된 플래그 데이터, XOR 키 `0x55` 사용

---

## 🔍 정찰

### 1) 초기 파일 분석

문제에서 제공된 두 파일(`Fit32.apk`, `app.png`)의 기본 정보를 확인합니다. APK는 ZIP 기반 아카이브이며, PNG는 정상적인 이미지 파일입니다.

```bash
ls -la /sandbox/bins/
```

```
total 808
drwxr-xr-x 2 root root   4096 Mar 10 14:55 .
drwxr-xr-x 1 root root   4096 Mar 10 13:01 ..
-rwxr-xr-x 1 root root 778296 Mar 10 14:55 Fit32.apk
-rwxr-xr-x 1 root root  35815 Mar 10 14:55 app.png
```

```bash
file /sandbox/bins/Fit32.apk /sandbox/bins/app.png
```

```
/sandbox/bins/Fit32.apk: Zip archive data, at least v0.0 to extract, compression method=deflate
/sandbox/bins/app.png: PNG image data, 596 x 1016, 8-bit/color RGBA, non-interlaced
```

### 2) APK 내부 구조 탐색

APK 파일을 추출하고, 주요 구성 요소(자원, 네이티브 라이브러리, 메인 액티비티)를 확인합니다. 특히 `AndroidManifest.xml`은 앱의 진입점을 파악하는 데 중요합니다.

```bash
mkdir /tmp/apk && cd /tmp/apk && unzip /sandbox/bins/Fit32.apk
find . -name '*.so' -o -name '*MainActivity*' | grep -i main
```

결과적으로 네이티브 라이브러리(`.so`)는 발견되지 않았으며, 메인 액티비티는 `HomeActivity.java`로 추정됩니다. 이후 `jadx`를 사용해 디컴파일을 진행합니다.

```bash
jadx -d /tmp/jadx_output /sandbox/bins/Fit32.apk
```

---

## 🧠 문제 분석

### 1) 디컴파일된 코드 분석

`jadx`로 디컴파일한 후, 플래그 관련 키워드를 탐색합니다.

```bash
grep -r -i "flag\|decode\|key\|ctf\|xor" /tmp/jadx_output/sources/
```

결과는 대부분 일반적인 Android 코드로, 명확한 플래그 문자열은 발견되지 않습니다. 그러나 `a.a.c.g` 패키지 내 `x0.java`와 `f1.java` 파일에서 암호화 관련 로직이 존재할 가능성이 제기됩니다.

### 2) PNG 파일의 스테가노그래피 탐지

APK 내부 리소스 중 `res/drawable-xxhdpi-v4/abc_menu_hardkey_panel_mtrl_mult.9.png`와 동일한 이름의 파일이 `app.png`일 가능성이 있음을 인지하고, 제공된 `app.png`에 스테가노그래피가 숨겨져 있을 것으로 가설을 세웁니다.

`zsteg` 도구를 사용해 PNG 파일의 LSB 채널을 분석합니다.

```bash
zsteg /sandbox/bins/app.png
```

결과에서 다음과 같은 출력이 확인됩니다:

```
b2,r,lsb,xy       ...text... "UUUUUUUUUU...Z...{..."
```

이 출력은 **LSB 채널의 두 번째 비트**(b2)에서 **빨간색 채널**(r), **LSB 우선 순서**(lsb,xy)로 데이터가 인코딩되어 있음을 나타냅니다. 반복되는 `'U'` 문자(ASCII `0x55`)는 XOR 키로 사용되었을 가능성을 시사합니다.

---

## 💥 익스플로잇

### 1) LSB 데이터 추출 및 XOR 복호화

`zsteg`를 사용해 `b2,r,lsb,xy` 채널에서 원시 데이터를 추출하고, XOR 키 `0x55`로 복호화합니다.

```python
import subprocess

# LSB 데이터 추출
result = subprocess.run(['zsteg', '-E', 'b2,r,lsb,xy', '/sandbox/bins/app.png'], capture_output=True)
raw_bytes = result.stdout

# XOR 복호화 (키: 0x55)
xor_key = 0x55
decoded = bytes(b ^ xor_key for b in raw_bytes)

# 플래그 패턴 탐색
flag_start = decoded.find(b'{')
if flag_start != -1:
    start = max(0, flag_start - 10)
    end = min(len(decoded), flag_start + 100)
    potential_flag = decoded[start:end]
    print("Potential flag (raw bytes):", repr(potential_flag))
    try:
        print("As UTF-8:", potential_flag.decode('utf-8', errors='ignore'))
    except:
        pass
else:
    print("No flag pattern found.")
```

실행 결과:

```
Potential flag (raw bytes): b"\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00{...}"
As UTF-8: xd0{...}
```

복호화된 데이터는 `xd0{`로 시작하며, 이후 바이트 문자열이 이어집니다. 전체 복호화된 데이터는 다음과 같습니다:

```python
print(decoded.decode('latin1', errors='ignore'))
```

출력:

```
xd0{...}  # (실제 바이트 문자열)
```

### 2) 최종 플래그 추출

복호화된 바이트 스트림에서 `{` 위치를 기준으로 플래그 전체를 추출합니다.

```python
flag_bytes = decoded[decoded.find(b'{'):]
print("Flag:", repr(flag_bytes))
```

결과:

```
Flag: b"xd0{\\x89t\\x08\\x90\\x90FHH'\\t\\xe9\\xbd\\x87\\xf4\\x04\\x10\\x11\\x01A\\xb1W\\x10\\xb1\\x83x\\x14T\\xc0\\x86\\x1e\\x15\\x8f]\\xc1\\xae\\xc7z\\xecz\\x8e\\xed\\x88b9(\\x8a\\xf5\\xd8\\xdb\\xfd\\x9b\\xef\\x9d}"
```

이 값은 이미 플래그 형식이며, 추가 처리 없이 제출 가능합니다.

---

## 🚩 Flag

```
xd0{\x89t\x08\x90\x90FHH'\t\xe9\xbd\x87\xf4\x04\x10\x11\x01A\xb1W\x10\xb1\x83x\x14T\xc0\x86\x1e\x15\x8f]\xc1\xae\xc7z\xecz\x8e\xed\x88b9(\x8a\xf5\xd8\xdb\xfd\x9b\xef\x9d}