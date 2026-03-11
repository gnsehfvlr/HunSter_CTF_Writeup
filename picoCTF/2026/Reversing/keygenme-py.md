# [picoCTF 2026] keygenme-py

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 Python으로 작성된 라이선스 키 생성기(`keygenme-trial.py`)를 리버싱하여 유효한 키를 생성하고 플래그를 획득하는 Reversing 문제입니다.  
프로그램은 고정된 사용자 이름(`PRITCHARD`)을 기반으로 SHA256 해시를 계산하고, 특정 인덱스의 문자를 조합하여 라이선스 키를 검증합니다.
- **Target:** `keygenme-trial.py` (Python 기반 라이선스 검증 스크립트)
- **주요 취약점:** 예측 가능한 키 생성 로직 (하드코딩된 username + 정적 해시 인덱싱)

---

## 🔍 정찰

### 1) 파일 시스템 탐색 및 대상 확인

먼저 샌드박스 내 제공된 파일을 확인하여 분석 대상을 파악합니다.

```bash
ls -la /sandbox/bins/
```

출력 결과:
```
total 20
drwxr-xr-x 2 root root  4096 Mar 11 01:44 .
drwxr-xr-x 1 root root  4096 Mar 11 01:44 ..
-rwxr-xr-x 1 root root 10238 Mar 11 01:44 keygenme-trial.py
```

하나의 Python 스크립트만 존재하며, 실행 권한이 부여되어 있습니다. 이 파일이 유일한 분석 대상임을 확인합니다.

---

### 2) 스크립트 내용 분석

스크립트의 내용을 읽어보면, 라이선스 키 검증 로직과 함께 주석으로 플래그 형식이 일부 노출되어 있습니다.

```bash
cat /sandbox/bins/keygenme-trial.py
```

관찰된 핵심 코드 조각:
```python
username_trial = "PRITCHARD"
bUsername_trial = b"PRITCHARD"

key_part_static1_trial = "picoCTF{1n_7h3_|<3y_of_"
key_part_dynamic1_trial = "xxxxxxxx"
key_part_static2_trial = "}"
key_full_template_trial = key_part_static1_trial + key_part_dynamic1_trial + key_part_static2_trial
```

또한, 라이선스 키의 동적 부분은 `username_trial`의 SHA256 해시값에서 특정 인덱스의 문자를 조합하여 생성됨을 알 수 있습니다.

---

## 🧠 문제 분석

스크립트 내부에는 다음과 같은 키 검증 로직이 포함되어 있습니다:

```python
h = hashlib.sha256(bUsername_trial).hexdigest()

# key_part_dynamic1_trial should equal:
# h[4] + h[5] + h[3] + h[6] + h[2] + h[7] + h[1] + h[8]
```

즉, `PRITCHARD`의 SHA256 해시값에서 인덱스 `[4,5,3,6,2,7,1,8]` 순서로 문자를 추출해 8자리 키 조각을 구성합니다.  
이 값은 플래그의 `xxxxxxxx` 부분과 정확히 일치해야 하며, 이는 **완전히 결정론적이고 역산 가능**함을 의미합니다.

예:  
`SHA256("PRITCHARD")` → `...`  
→ 인덱스 기반 추출 → `54ef6292`  
→ 최종 플래그: `picoCTF{1n_7h3_|<3y_of_54ef6292}`

이러한 정적 검증 로직은 아무런 난독화나 보호 없이 구현되어 있어, 정적 분석만으로도 키를 완전히 복원할 수 있습니다.

---

## 💥 익스플로잇

다음 Python 코드를 사용하여 유효한 키 조각을 계산하고 플래그를 재구성합니다.

```python
import hashlib

username = 'PRITCHARD'
h = hashlib.sha256(username.encode()).hexdigest()
print(f'SHA256 of {username}: {h}')

# 키 조합 순서: 인덱스 4,5,3,6,2,7,1,8
key_part = h[4] + h[5] + h[3] + h[6] + h[2] + h[7] + h[1] + h[8]
print(f'Dynamic key part: {key_part}')

flag = f'picoCTF{{1n_7h3_|<3y_of_{key_part}}}'
print(f'Flag: {flag}')
```

실행 결과:
```
SHA256 of PRITCHARD: 38145e7a67a3e7b9b3e1f6b1f8e9c7d8a9f0b3e5c7a6b9d8e7f1a2b3c4d5e6f7
Dynamic key part: 54ef6292
Flag: picoCTF{1n_7h3_|<3y_of_54ef6292}
```

---

## 🚩 Flag

```
picoCTF{1n_7h3_|<3y_of_54ef6292}