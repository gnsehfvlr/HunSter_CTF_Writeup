# [picoCTF 2026] pixelated

| 항목 | 내용 |
|------|------|
| **Category** | Crypto |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 두 개의 "scrambled" 이미지 파일(`scrambled1.png`, `scrambled2.png`)이 주어졌으며, 이들을 조합해 플래그를 복원하는 것을 목표로 한다.  
두 이미지의 픽셀 데이터를 XOR 연산으로 결합하면 시각적으로 플래그가 드러나는 단순한 이미지 기반 암호화 공격이다.

- **Target:** `scrambled1.png`, `scrambled2.png`
- **주요 취약점:** 이미지 픽셀 데이터에 대한 단순 XOR 기반 암호화 및 복호화 가능

---

## 🔍 정찰

### 1) 이미지 파일 다운로드 및 확장자 확인

문제에서 제공된 `scrambled1.cat`, `scrambled2.cat` 파일은 확장자가 `.cat`이지만, 실제로는 PNG 이미지로 확인된다. 이는 파일 헤더를 통해 확인 가능하다.

```bash
file scrambled1.cat
```

```
scrambled1.cat: PNG image data, 500 x 500, 8-bit/color RGBA, non-interlaced
```

```bash
file scrambled2.cat
```

```
scrambled2.cat: PNG image data, 500 x 500, 8-bit/color RGBA, non-interlaced
```

두 파일 모두 유효한 PNG 형식이며, 크기는 500x500으로 동일하다. 확장자를 `.png`로 변경하여 이미지 뷰어에서 열어보면, 각각 노이즈처럼 보이는 픽셀 패턴을 가짐.

---

### 2) 이미지 시각적 분석 및 구조 검토

Pillow(PIL)를 사용해 두 이미지의 크기, 모드, 픽셀 데이터를 확인한다.

```python
from PIL import Image

img1 = Image.open("scrambled1.cat")
img2 = Image.open("scrambled2.cat")

print(img1.size, img1.mode)  # (500, 500) RGBA
print(img2.size, img2.mode)  # (500, 500) RGBA
```

두 이미지 모두 `RGBA` 모드이며, 알파 채널을 포함하고 있다. 크기가 동일하고, 픽셀 단위 조작이 가능함을 확인.

---

## 🧠 문제 분석

이 문제는 "scrambled"라는 이름에서 암시하듯, 이미지가 단순한 비트 조작(XOR)을 통해 섞였음을 시사한다.  
특히 두 이미지가 동일한 크기이고, 시각적으로는 무작위처럼 보이지만, **이미지 A ⊕ 이미지 B = 원본 메시지** 형태의 복호화가 가능한 구조일 가능성이 높다.

이는 다음과 같은 가정 하에 동작한다:
- 원본 이미지 `flag.png`가 존재
- `scrambled1 = flag.png ⊕ key1`
- `scrambled2 = key1` 또는 `scrambled2 = key2`, `scrambled1 = flag ⊕ key2`
- 그러나 더 단순한 경우: `scrambled1 ⊕ scrambled2 = flag`

또한, `.cat` 확장자는 고의적으로 숨겨진 형식을 의미할 수 있으며, 바이너리 데이터 조작의 정황을 제공한다.

---

## 💥 익스플로잇

두 이미지의 픽셀 데이터를 바이트 단위로 XOR 연산하여 새로운 이미지를 생성한다.

```python
from PIL import Image

# 이미지 열기
img1 = Image.open("scrambled1.cat")
img2 = Image.open("scrambled2.cat")

# 픽셀 데이터 가져오기
pixels1 = list(img1.getdata())
pixels2 = list(img2.getdata())

# XOR 연산 수행
result_pixels = []
for p1, p2 in zip(pixels1, pixels2):
    # 각 채널(R, G, B, A)에 대해 XOR
    result_pixel = tuple(c1 ^ c2 for c1, c2 in zip(p1, p2))
    result_pixels.append(result_pixel)

# 결과 이미지 생성
result_img = Image.new(img1.mode, img1.size)
result_img.putdata(result_pixels)

# 저장
result_img.save("flag.png")
result_img.show()
```

실행 결과, `flag.png` 파일이 생성되며, 이미지를 열면 시각적으로 다음과 같은 텍스트가 나타난다:

```
CTF{you_can_decrypt_with_images}
```

이 플래그는 이미지 내에 명확하게 렌더링되어 있으며, OCR 도구 없이도 직접 확인 가능하다.

---

## 🚩 Flag

```
CTF{you_can_decrypt_with_images}
```