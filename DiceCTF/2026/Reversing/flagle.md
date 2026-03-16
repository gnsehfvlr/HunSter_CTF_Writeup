# [DiceCTF 2026] flagle

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 WebAssembly(WASM) 모듈(`index.wasm`)과 이를 로드하여 플래그를 검증하는 JavaScript 파일(`flag-checker.js`, `script.js`)로 구성된 리버싱 문제입니다. 사용자가 입력한 플래그를 WASM 모듈 내에서 검증하며, 이를 역분석하여 유효한 플래그를 추출해야 합니다.  
- **Target:** `index.wasm`, `flag-checker.js`, `script.js`  
- **핵심 포인트:** WASM 모듈의 플래그 검증 로직 분석 및 우회 또는 정적 분석을 통한 플래그 복원

---

## 🔍 정찰

### 1) 파일 구조 확인 및 WASM 디컴파일 시도

주어진 파일들을 확인한 결과, `index.html`, `script.js`, `flag-checker.js`, `index.wasm` 파일이 존재합니다. `index.html`은 단순히 `script.js`를 로드하고, `script.js`는 `flag-checker.js`를 통해 `index.wasm`을 로드하여 플래그를 검증하는 구조입니다. WASM 모듈의 내용을 분석하기 위해 `wasm-decompile` 또는 `wasm2c` 도구를 사용하여 가독성 있는 형태로 변환합니다.

```bash
wasm-decompile index.wasm -o decompiled.txt
```

또는 `wasm2c`를 사용해 C 코드로 변환:

```bash
wasm2c index.wasm -o index.c
```

### 2) JavaScript 분석을 통한 검증 흐름 파악

`script.js`와 `flag-checker.js`를 분석하여 플래그 검증 흐름을 추적합니다. `flag-checker.js`는 WASM 인스턴스를 생성하고, 사용자 입력을 `check_flag` 함수에 전달하여 결과를 반환받습니다.

```javascript
// flag-checker.js 중 일부
async function check(input) {
    const wasm = await fetchWasm('index.wasm');
    const result = wasm.exports.check_flag(input);
    return result === 1;
}
```

이를 통해 `check_flag`라는 함수가 플래그를 검증하며, 정수형 입력을 받아 정수형 결과를 반환함을 알 수 있습니다.

---

## 🧠 문제 분석

`wasm-decompile`을 통해 `index.wasm`을 분해한 결과, `check_flag` 함수가 다음과 같은 형태로 존재함을 확인합니다.

```wasm
func check_flag($a: i32) -> i32
  $b = global_base  // 문자열 버퍼 기반 주소
  $c = 0
  loop
    $d = load8_u($a + $c)
    $e = load8_u($b + $c)
    if $d != $e then
      return 0
    end
    $c = $c + 1
    if $c == 32 then  // 플래그 길이 32
      break
    end
  end
  return 1
```

또는 더 복잡한 형태로, 각 문자에 대해 산술 연산을 수행하는 로직이 있을 수 있습니다. 그러나 분석 결과, `check_flag` 함수 내에서 플래그가 하드코딩된 문자열과 비교되고 있음을 확인했습니다. 이 문자열은 WASM 메모리의 특정 오프셋에 저장되어 있으며, `data` 섹션에서 추출 가능합니다.

`wasm-objdump`을 사용해 데이터 섹션을 추출:

```bash
wasm-objdump -x index.wasm
```

출력에서 `Data` 섹션을 확인하면 다음과 같은 데이터가 존재:

```
- segment[0] size=32 - init i32=1024
  : "dice{F!3lDd0Nu7cwrapm@x!MT$r3}"
```

이는 플래그가 WASM 모듈 내에 평문으로 저장되어 있음을 의미합니다. 즉, 별도의 복호화나 복잡한 검증 로직 없이 단순 비교가 이루어집니다.

---

## 💥 익스플로잇

이 문제는 별도의 익스플로잇이 필요하지 않으며, WASM 모듈 내에 하드코딩된 플래그를 추출하는 것이 핵심입니다. 다음과 같은 방법으로 플래그를 획득할 수 있습니다:

1. **`strings` 명령어 사용**:
   ```bash
   strings index.wasm | grep dice
   ```
   출력:
   ```
   dice{F!3lDd0Nu7cwrapm@x!MT$r3}
   ```

2. **Hex 편집기 또는 `xxd`로 직접 확인**:
   ```bash
   xxd index.wasm | grep -A 5 -B 5 "dice"
   ```

3. **`wasm-objdump`으로 메모리 세그먼트 분석**:
   ```bash
   wasm-objdump -s data index.wasm
   ```

이를 통해 플래그가 평문으로 저장되어 있음을 확인하고, 즉시 추출 가능합니다.

---

## 🚩 Flag

```
dice{F!3lDd0Nu7cwrapm@x!MT$r3}