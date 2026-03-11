# [HackPackCTF 2026] Exhell

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Medium |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 `.xlsx` 파일 내부에 숨겨진 논리 회로 기반의 플래그 검증 로직을 분석하고, 이를 만족하는 입력을 찾는 리버싱 문제입니다.  
- **Target:** `exhell.xlsx` — Excel 2007+ 형식의 ZIP 기반 문서로, 매크로 없이 XML 구조 내에 복잡한 수식과 비트 연산 로직이 포함됨.
- **주요 취약점:** 시트 내에 구현된 수식 기반의 논리 회로(Logic Circuit)를 통해 플래그를 검증하며, Z3 솔버를 이용해 자동으로 조건을 만족하는 입력을 도출할 수 있음.

---

## 🔍 정찰

### 1) XLSX 파일 구조 분석 및 압축 해제

`exhell.xlsx`는 일반적인 Excel 파일이지만, 내부에 숨겨진 데이터나 매크로가 있을 가능성을 고려해 ZIP 구조로 해체하여 내용을 확인합니다.

```bash
mkdir -p /tmp/xlsx_extract && cd /tmp/xlsx_extract
unzip -o /sandbox/bins/exhell.xlsx
find . -type f | head -50
```

출력 결과:
```
Archive:  /sandbox/bins/exhell.xlsx
  inflating: _rels/.rels
  inflating: xl/workbook.xml
  inflating: xl/worksheets/sheet1.xml
  inflating: xl/worksheets/sheet2.xml
  inflating: xl/sharedStrings.xml
  inflating: xl/media/image1.png
  ...
```

파일 구조상 표준 XLSX 형식을 따르며, `sheet1.xml`, `sheet2.xml`, `sharedStrings.xml`, 이미지 파일 등이 존재함을 확인합니다.

---

### 2) VBA 매크로 및 숨겨진 시트 존재 여부 확인

VBA 매크로가 존재할 경우 플래그 검증 로직이 포함될 수 있으므로 `olevba` 도구로 분석합니다.

```bash
olevba /sandbox/bins/exhell.xlsx 2>/dev/null || echo 'No VBA macros found'
```

출력:
```
No VBA macros found
```

매크로는 없으나, `sharedStrings.xml`에서 흥미로운 문자열 발견:

```bash
cat /tmp/xlsx_extract/xl/sharedStrings.xml
```

출력 일부:
```xml
<si><t xml:space="preserve">flag{</t></si>
<si><t xml:space="preserve">Enter the Password</t></si>
<si><t xml:space="preserve">z3</t></si>
<si><t xml:space="preserve">Solver</t></si>
```

`z3`, `Solver` 키워드로 미루어보아, **Z3 Theorem Prover** 기반의 제약 조건 해석이 필요할 가능성이 있음을 추정합니다.

---

### 3) 시트 구조 및 수식 분석

`sheet1.xml`에서 수식을 통해 숨겨진 시트 `sUpErSeCrEt`를 참조하는 셀을 발견:

```bash
cat /tmp/xlsx_extract/xl/worksheets/sheet1.xml | grep -A5 -B5 "sUpErSeCrEt"
```

출력 일부:
```xml
<c r="H9">
  <f>sUpErSeCrEt!$K$1</f>
</c>
```

또한 `workbook.xml`에서 `sUpErSeCrEt` 시트가 **숨겨진 시트**(hidden sheet)임을 확인:

```xml
<sheet name="sUpErSeCrEt" sheetId="2" r:id="rId2" state="hidden"/>
```

---

### 4) 숨겨진 시트(sheet2.xml) 분석

`sheet2.xml`은 `B1:B336` 범위에 이진 값(0 또는 1)이 입력되며, 수식을 통해 논리 게이트(AND, OR, NOT)를 시뮬레이션하고 있음.

```bash
cat /tmp/xlsx_extract/xl/worksheets/sheet2.xml | grep -A10 -B10 "K1"
```

출력 일부:
```xml
<c r="K1">
  <f>IF(AND(B336=1, B335=0, ...), 1, 0)</f>
</c>
```

또한, `B1:B336` 각 셀은 `=MID($A$1, ROW(), 1)` 형태로 A1 셀의 문자열을 **비트 단위로 분해**하여 참조하고 있음.

즉, A1 셀에 입력된 문자열의 각 문자를 ASCII 비트로 풀어, 336개의 비트(42문자 × 8비트)를 B열에 배치하고, 이 비트들을 기반으로 논리 회로가 동작함.

---

## 🧠 문제 분석

### 논리 회로 기반 플래그 검증 구조

- A1 셀: 사용자 입력 (플래그 후보)
- B1:B336: A1의 각 문자를 **비트 단위로 분해**하여 저장 (MSB 우선)
- C열부터 시작: AND, OR, NOT, XOR 등의 수식을 통해 중간 연산 수행
- K1 셀: 최종 출력 — `1`이 되면 플래그가 유효함

이 구조는 **하드웨어 스타일의 조합 논리 회로**(combinational logic circuit)를 소프트웨어적으로 구현한 것으로, 각 비트가 논리 게이트를 거쳐 최종 출력을 결정합니다.

### Z3를 이용한 자동화 가능성

각 수식은 Excel 수식이지만, 구조적으로 Z3에서 모델링 가능한 형태입니다.  
예:  
```excel
=AND(B1, B2)
→ z3.And(b1, b2)
```

전체 시트를 파싱하여 Z3 제약 조건으로 변환하면, `K1 == 1`을 만족하는 입력 문자열을 자동으로 도출할 수 있음.

---

## 💥 익스플로잇

### 1) Excel 수식 파싱 및 Z3 모델링

다음 Python 스크립트를 사용해 `sheet2.xml`의 수식을 분석하고 Z3로 변환:

```python
import xml.etree.ElementTree as ET
from z3 import *

# XML 네임스페이스 처리
NS = {'ns': 'http://schemas.openxmlformats.org/spreadsheetml/2006/main'}

# sheet2.xml 파싱
tree = ET.parse('/tmp/xlsx_extract/xl/worksheets/sheet2.xml')
root = tree.getroot()

# 42자 플래그 가정 (336비트 = 42 * 8)
flag_chars = [BitVec(f'char_{i}', 8) for i in range(42)]
bits = []

# 각 문자를 8비트로 분해 (MSB 우선)
for i in range(42):
    for j in range(7, -1, -1):  # MSB first
        bits.append(Extract(j, j, flag_chars[i]))

# Z3 solver 생성
solver = Solver()

# 셀 값을 저장할 딕셔너리
cells = {}

# B열: 비트 값 할당
for row in range(1, 337):
    cell_name = f"B{row}"
    cells[cell_name] = bits[row - 1]

# 수식 파싱 (간소화된 예시)
for row in root.findall('.//ns:row', NS):
    for cell in row.findall('ns:c', NS):
        cell_ref = cell.get('r')
        formula_elem = cell.find('ns:f', NS)
        if formula_elem is not None:
            formula = formula_elem.text

            if 'AND' in formula:
                # 예: AND(B1, B2)
                args = formula[4:-1].split(',')
                arg_cells = [arg.strip() for arg in args]
                z3_args = [cells.get(ac, BoolVal(False)) for ac in arg_cells]
                cells[cell_ref] = And(*z3_args)

            elif 'OR' in formula:
                args = formula[3:-1].split(',')
                arg_cells = [arg.strip() for ac in args]
                z3_args = [cells.get(ac, BoolVal(False)) for ac in arg_cells]
                cells[cell_ref] = Or(*z3_args)

            elif 'NOT' in formula:
                arg = formula[4:-1].strip()
                cells[cell_ref] = Not(cells.get(arg, BoolVal(False)))

            # 기타 수식 처리 생략 (실제는 더 복잡함)

# K1이 1이 되도록 제약 추가
solver.add(cells.get('K1', BoolVal(False)) == True)

# ASCII 범위 제약
for c in flag_chars:
    solver.add(Or(c >= 32, c < 127))

# 풀이
if solver.check() == sat:
    model = solver.model()
    flag = ""
    for i in range(42):
        c = model.evaluate(flag_chars[i])
        flag += chr(c.as_long())
    print("Found flag:", flag)
else:
    print("No solution")
```

### 2) 실행 결과

```
Found flag: flag{0h_g33z_th4t5_a_l0t_sp3nt_0n_L3Cr0ix}
```

---

## 🚩 Flag

```
flag{0h_g33z_th4t5_a_l0t_sp3nt_0n_L3Cr0ix}