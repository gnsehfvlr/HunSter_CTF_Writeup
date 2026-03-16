# [CyberGon 2026] Hollywood

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 주어진 파일들(`Nuke.zip`, `breakpoint.png`, `obfuscated.png`)을 분석하여 숨겨진 플래그를 추출하는 리버싱 문제입니다. 이미지 파일 내부에 난독화된 데이터 또는 숨겨진 코드가 포함되어 있으며, 이를 복호화하거나 추출하는 과정이 필요합니다.  
- **Target:** `obfuscated.png`, `breakpoint.png`, `Nuke.zip`  
- **핵심 포인트:** 난독화된 이미지에서 숨겨진 데이터 추출 및 복호화

---

## 🔍 정찰

### 1) 파일 구조 분석

주어진 파일들을 확인하기 위해 `file` 명령어를 사용하여 각 파일의 형식을 분석했습니다.

```bash
file Nuke.zip breakpoint.png obfuscated.png
```

결과:
```
Nuke.zip:        Zip archive data, at least v2.0 to extract
breakpoint.png:  PNG image data, 500 x 500, 8-bit/color RGBA, non-interlaced
obfuscated.png:  PNG image data, 1000 x 1000, 8-bit/color RGBA, non-interlaced
```

`Nuke.zip`이 존재하므로 압축을 해제해 보았습니다.

```bash
unzip Nuke.zip
```

압축 해제 후, `script.py`라는 파이썬 파일이 추출되었습니다.

---

### 2) 스크립트 분석 (`script.py`)

`script.py` 파일을 열어 내용을 확인했습니다. 이 스크립트는 `obfuscated.png`를 기반으로 플래그를 복원하는 역할을 하는 것으로 보입니다.

```python
from PIL import Image

def main():
    img = Image.open("obfuscated.png")
    pixels = img.load()
    width, height = img.size

    flag = ""
    for y in range(0, height, 10):
        for x in range(0, width, 10):
            r, g, b, a = pixels[x, y]
            if r == 255 and g == 0 and b == 0:  # Red pixel
                flag += chr((x * y) % 256)
            elif r == 0 and g == 255 and b == 0:  # Green pixel
                flag += chr((x + y) % 256)
            elif r == 0 and g == 0 and b == 255:  # Blue pixel
                flag += chr((x ^ y) % 256)

    print(flag)

if __name__ == "__main__":
    main()
```

이 스크립트는 특정 간격(10픽셀)으로 픽셀을 읽고, RGB 값에 따라 다른 방식으로 문자를 생성합니다:
- 빨간 픽셀: `(x * y) % 256`
- 초록 픽셀: `(x + y) % 256`
- 파란 픽셀: `(x ^ y) % 256`

---

## 🧠 문제 분석

`obfuscated.png`는 시각적으로는 무작위 색상의 점들로 보이지만, 실제로는 픽셀의 위치와 색상 정보를 이용해 문자열을 인코딩한 것으로 보입니다.  
또한, `breakpoint.png`는 문제의 힌트를 제공할 수 있는 파일로, 메타데이터를 확인해 보았습니다.

```bash
exiftool breakpoint.png
```

결과에서 다음과 같은 주석이 발견되었습니다:
```
Comment                         : Look at red, green, blue. Order matters.
```

이 힌트는 픽셀의 색상 순서가 중요하다는 것을 시사하며, `script.py`의 로직과 일치합니다.

또한, `script.py`는 `obfuscated.png`에서 10픽셀 간격으로만 데이터를 읽고 있으므로, 이 간격이 정확히 맞아야 합니다.

---

## 💥 익스플로잇

`script.py`를 실행하여 `obfuscated.png`에서 플래그를 추출합니다. 단, 현재 스크립트는 `print(flag)`만 하므로, 출력 결과를 확인합니다.

```python
# script.py 실행
python3 script.py
```

실행 결과:
```
h4ck!ng_!n_h00lyw00d_$33m_c00l_unl!k3_r34l_w0rld!
```

스크립트가 정상적으로 플래그를 출력합니다. 이는 픽셀 기반 인코딩 방식을 통해 문자열이 숨겨져 있었으며, 스크립트가 이를 정확히 디코딩했음을 의미합니다.

---

## 🚩 Flag

```
h4ck!ng_!n_h00lyw00d_$33m_c00l_unl!k3_r34l_w0rld!
```