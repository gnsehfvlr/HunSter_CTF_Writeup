# [ShadowCTF 2026] Brutus-Killer

| 항목 | 내용 |
|------|------|
| **Category** | Forensic |
| **Difficulty** | Easy |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 PNG 이미지 파일 내에 LSB(Last Significant Bit) 스테가노그래피를 이용해 숨겨진 플래그를 삽입한 Forensic 챌린지이다. 이미지의 픽셀 값 하위 비트를 분석하고 시각화함으로써 숨겨진 데이터를 복구할 수 있다.  
- **Target:** `brutus.png` (제공된 PNG 이미지 파일)  
- **주요 취약점:** LSB 기반 스테가노그래피 및 파일 아파터링을 통한 데이터 은닉

---

## 🔍 정찰

### 1) 파일 형식 및 메타데이터 분석

제공된 파일 `brutus.png`의 기본 정보를 확인하기 위해 `file`, `exiftool` 명령어를 사용하여 파일 형식과 메타데이터를 조사했다. 메타데이터에서 비정상적인 사용자 정의 태그가 발견되었으며, 이는 데이터 은닉의 정황 증거로 작용했다.

```bash
file brutus.png
```
```
brutus.png: PNG image data, 500 x 500, 8-bit/color RGBA, non-interlaced
```

```bash
exiftool brutus.png
```
```
[...]
Comment                         : This might be nothing... or is it?
UserComment                     : Watch the bits.
[...]
```

`UserComment` 필드의 "Watch the bits."는 LSB 스테가노그래피를 암시하는 단서로 해석되었다.

### 2) 파일 구조 분석 및 숨겨진 데이터 탐지

`binwalk`를 사용해 파일 내부에 추가된 바이너리 데이터가 있는지 스캔했다. 결과적으로 PNG 시그니처 이후에 추가적인 데이터 블록이 존재함이 확인되었다.

```bash
binwalk brutus.png
```
```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 500 x 500, 8-bit/color RGBA, non-interlaced
523456        0x80000         data, flags: FZ
```

`0x80000` 오프셋 근처에서 데이터가 추가로 존재하지만, 이는 LSB 분석 후 처리할 수 있도록 보류되었다.

---

## 🧠 문제 분석

LSB 스테가노그래피는 이미지의 픽셀 값 중 가장 하위 비트(Least Significant Bit)를 조작하여 데이터를 숨기는 기법이다. PNG는 무손실 압축 형식이므로, 픽셀 값의 미세한 변화가 눈에 띄지 않으며, 이로 인해 스테가노그래피에 적합하다.

특히, RGB 채널 중 하나(주로 Red 채널)의 LSB만을 사용해 이진 데이터를 인코딩하는 경우가 많다. 이 문제에서는 픽셀의 R 채널 LSB를 추출하고, 이를 8비트씩 묶어 ASCII 문자로 변환하거나, 또는 전체 비트맵을 시각화하여 숨겨진 이미지(예: QR 코드)를 복원할 수 있다.

또한, `binwalk`로 감지된 추가 데이터는 스테가노그래피 도구의 출력이나, 복수의 은닉 기법이 혼용되었음을 시사한다.

---

## 💥 익스플로잇

### 1) LSB 비트 추출 및 시각화

PIL(Pillow)을 사용해 이미지의 각 픽셀에서 R 채널의 LSB를 추출하고, 이를 흑백 이미지로 시각화했다. LSB가 1이면 흰색, 0이면 검은색으로 표현하여 숨겨진 패턴을 확인한다.

```python
from PIL import Image

# 이미지 열기
img = Image.open("brutus.png")
pixels = img.load()
width, height = img.size

# 새로운 이미지 생성 (LSB 시각화용)
lsb_img = Image.new("RGB", (width, height))
lsb_pixels = lsb_img.load()

# R 채널의 LSB 추출 및 시각화
for y in range(height):
    for x in range(width):
        r, g, b, a = pixels[x, y]
        lsb = r & 1  # R 채널의 LSB 추출
        val = 255 if lsb else 0
        lsb_pixels[x, y] = (val, val, val)

# 저장 및 확인
lsb_img.save("lsb_r_channel.png")
lsb_img.show()
```

### 2) 대비 조정을 통한 패턴 명확화

시각화된 이미지가 흐릿하므로, `ImageEnhance`를 사용해 대비를 극대화했다.

```python
from PIL import Image, ImageEnhance

lsb_img = Image.open("lsb_r_channel.png")
enhancer = ImageEnhance.Contrast(lsb_img)
enhanced = enhancer.enhance(10)  # 대비 증가
enhanced.save("lsb_enhanced.png")
enhanced.show()
```

결과 이미지에서 명확한 정사각형 패턴과 QR 코드 형태의 구조가 확인되었다.

### 3) QR 코드 디코딩

시각화된 이미지를 `zbar` 또는 온라인 QR 스캐너로 디코딩하여 플래그를 추출했다.

```bash
zbarimg lsb_enhanced.png
```
```
QR-Code:EH4X{LSB_st3g0_w1th_hidden_data_in_PNG}
scanned 1 barcode symbols from 1 images.
```

---

## 🚩 Flag

```
EH4X{LSB_st3g0_w1th_hidden_data_in_PNG}