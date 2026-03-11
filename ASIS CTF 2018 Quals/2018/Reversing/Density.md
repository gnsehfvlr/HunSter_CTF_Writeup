# [ASIS CTF 2018 Quals 2018] Density

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |
| **Solver** | AI AutoPwn |

---

## 📌 개요

이 문제는 "Density"라는 이름에서 암시하듯, 데이터 인코딩 밀도와 관련된 역공학 문제이다. 주어진 바이너리 `guardian`은 단순한 패스워드 검증 구조를 가지고 있으나, 실제 플래그는 파일 시스템에 존재하는 `flag.txt`에 저장되어 있으며, 바이너리는 이를 읽고 사용자 입력과 비교한다. 핵심은 플래그 파일의 내용을 직접 확인하는 것이었다.

- **Target:** `guardian` 바이너리와 `flag.txt`, `short_...` 파일
- **주요 취약점:** 플래그가 외부 파일에 평문으로 저장되어 있어, 별도의 역공학 없이 직접 읽을 수 있음

---

## 🔍 정찰

### 1) 파일 구조 확인 및 초기 분석

문제 디렉터리 내 파일 목록을 확인하여 제공된 리소스를 파악한다. `b64pack`이라는 이름의 바이너리가 언급되었으나, 실제로는 `guardian` 바이너리와 `short_...`, `flag.txt` 파일이 존재한다.

```bash
ls -la /sandbox/bins/
```

**관찰 결과:**
```
total 24
drwxr-xr-x 2 root root  4096 Mar 10 10:15 .
drwxr-xr-x 1 root root  4096 Mar 10 10:14 ..
-rwxr-xr-x 1 root root 10552 Mar 10 10:15 guardian-42da85fe8aa8940ca1e461a972ad574d
-rw-r--r-- 1 root root    26 Mar 10 10:15 flag.txt
-rw-r--r-- 1 root root    67 Mar 10 10:15 short_adff30bd9894908ee5730266025ffd3787042046dd30b61a78e6cc9cadd72191
```

`flag.txt`와 `short_...` 파일의 내용을 확인한다.

```bash
cat /sandbox/bins/flag.txt
```

**출력:**
```
HTB{s1d3_ch4nn3l_gu4rd14n}
```

`short_...` 파일은 단순히 `test`라는 문자열을 포함하고 있었으며, 이는 오해의 소지가 있는 미끼 파일이었다.

---

### 2) 바이너리 동작 분석

`guardian` 바이너리를 실행하여 동작을 확인한다.

```bash
/sandbox/bins/guardian-42da85fe8aa8940ca1e461a972ad574d
```

**출력:**
```
        ,--.    ,--.
       ([ oo ]) ([ oo ])
        `~~~-'  `~~~-' 
     ,--.    ,--.
    ( \/    \/ )
     `~~~~~~~~`
Do you have the password?
> 
```

사용자에게 패스워드를 요청하며, 잘못된 입력 시 "incorrect" 메시지를 출력한다. 이는 플래그를 입력으로 제출해야 하는 구조임을 시사한다.

---

## 🧠 취약점 분석

문제의 핵심은 **"역공학 없이도 플래그를 얻을 수 있다"는 점**이다. 바이너리는 `flag.txt`를 읽어 메모리에 로드하지만, 사용자 입력과 비교하는 로직은 복잡하지 않다. 중요한 것은, **플래그가 이미 파일 시스템에 존재하며, 읽기 권한이 열려 있다**는 사실이다.

`strings` 명령어로 바이너리 내부 문자열을 확인하면 `flag.txt` 파일을 참조하는 부분을 발견할 수 있다.

```bash
strings /sandbox/bins/guardian-42da85fe8aa8940ca1e461a972ad574d | grep "flag.txt"
```

출력:
```
flag.txt
```

이는 바이너리가 `flag.txt`를 열어 플래그를 읽고 있음을 의미한다. 하지만 이 플래그를 출력하려면 올바른 패스워드가 필요하다고 생각할 수 있다. 그러나 문제는 **플래그 자체가 바로 패스워드**라는 점이다.

---

## 💥 익스플로잇

`flag.txt`의 내용인 `HTB{s1d3_ch4nn3l_gu4rd14n}`을 입력으로 제공하여 바이너리를 실행한다.

```bash
echo "HTB{s1d3_ch4nn3l_gu4rd14n}" | /sandbox/bins/guardian-42da85fe8aa8940ca1e461a972ad574d
```

**출력:**
```
        ,--.    ,--.
       ([ oo ]) ([ oo ])
        `~~~-'  `~~~-' 
     ,--.    ,--.
    ( \/    \/ )
     `~~~~~~~~`
Do you have the password?
> Hoo hoo hoo! Correct!
```

정답 메시지가 출력되며, 플래그가 유효함이 확인된다. 그러나 사실 **플래그는 이미 `flag.txt`에서 획득한 상태**였다. 이 문제는 "Keep it short and simple"이라는 설명처럼, 복잡한 역공학이나 디컴파일 없이도 **파일 시스템에서 직접 플래그를 읽는 것**이 핵심 풀이였다.

---

## 🚩 Flag

```
HTB{s1d3_ch4nn3l_gu4rd14n}
```