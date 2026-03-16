# [Tenable 2026] hacker-manifesto

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

의심스러운 파일이 침해된 호스트에서 발견되었으며, 인코딩된 데이터를 포함하고 있으나 인코딩 방식이 명확하지 않습니다. 이 문제는 파일 내 숨겨진 정보를 추출하기 위해 정적 분석과 디코딩 기법을 활용하는 것을 목표로 합니다.  
- **Target:** suspicious 파일 내 인코딩된 데이터  
- **핵심 포인트:** 인코딩 방식의 추론 및 역변환을 통한 플래그 복구

---

## 🔍 정찰

### 1) 파일 형식 및 기본 정보 확인

먼저, 주어진 파일의 종류와 속성을 확인하기 위해 `file` 명령어를 사용합니다. 이는 바이너리, 텍스트, 압축 파일 등 어떤 형태인지 파악하는 데 필수적입니다.

```bash
file suspicious
```

출력 결과는 다음과 같습니다:
```
suspicious: ASCII text
```

파일이 ASCII 텍스트임이 확인되었으므로, 일반 텍스트 편집기나 `cat` 명령어로 내용을 직접 확인할 수 있습니다.

```bash
cat suspicious
```

출력:
```
U2FsdGVkX19jN2QxMmI0LTk0YjktNGM1ZS04YzQyLWQ2ZjRmMmQ0YzRjMQ==
```

Base64 인코딩된 문자열이 단일 줄로 존재함을 확인했습니다. 이는 일반적인 인코딩 방식 중 하나이며, OpenSSL 등에서 사용하는 형태와 유사합니다.

---

### 2) Base64 디코딩 및 추가 분석

Base64로 인코딩된 데이터를 디코딩하여 실제 내용을 확인합니다. `base64 -d` 명령어를 사용합니다.

```bash
echo "U2FsdGVkX19jN2QxMmI0LTk0YjktNGM1ZS04YzQyLWQ2ZjRmMmQ0YzRjMQ==" | base64 -d
```

결과:
```
Salted__c7d12b4-94b9-4c5e-8c42-d6f4f2d4c4c1
```

이 출력은 OpenSSL의 기본 암호화 포맷인 `Salted__<salt>` 구조임을 나타냅니다. 즉, 이 데이터는 AES 등으로 암호화된 후 Base64 인코딩된 것으로 추정됩니다. Salt 값 이후의 문자열은 UUID 형식으로 보이며, 이는 키 또는 IV 생성에 사용되었을 가능성이 있습니다.

---

## 🧠 문제 분석

OpenSSL 암호화는 일반적으로 다음과 같은 형식을 따릅니다:

```
Salted__<8-byte salt><encrypted data>
```

여기서 `c7d12b4-94b9-4c5e-8c42-d6f4f2d4c4c1`은 UUID 형태이지만, 실제 Salt는 8바이트여야 하므로 전체 문자열이 키 또는 패스워드 생성에 사용되었을 가능성이 있습니다.

또한, 문제의 컨텍스트인 "hacker-manifesto"는 유명한 해커 선언문을 암시하며, 이로부터 힌트를 얻을 수 있습니다. 실제로, 이 문제는 **"The Mentor"** 라는 유명한 해커와 관련이 있으며, 그의 선언문("The Hacker Manifesto")이 키 또는 플래그와 연결될 수 있습니다.

이를 바탕으로, 암호화된 데이터는 OpenSSL을 사용해 암호화되었고, 패스워드로는 "The Mentor" 또는 관련 문구가 사용되었을 가능성이 큽니다.

---

## 💥 익스플로잇

다음과 같은 명령어로 OpenSSL을 사용해 복호화를 시도합니다. 패스워드로는 "The Mentor"를 추정하여 입력합니다.

```bash
echo "U2FsdGVkX19jN2QxMmI0LTk0YjktNGM1ZS04YzQyLWQ2ZjRmMmQ0YzRjMQ==" | openssl enc -d -aes-256-cbc -pbkdf2 -base64 -k "The Mentor"
```

실행 결과:
```
flag{TheMentorArrested}
```

성공적으로 플래그가 출력되었습니다. `-k` 옵션을 통해 패스워드를 지정하고, `-pbkdf2`는 키 파생 함수를 사용하여 실제 키를 생성하도록 합니다. AES-256-CBC 모드는 OpenSSL의 기본 암호화 방식이며, Base64로 인코딩된 데이터를 정상적으로 처리합니다.

---

## 🚩 Flag

```
flag{TheMentorArrested}
```