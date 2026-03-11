# [ASIS CTF 2018 Quals 2018] Heavy_makeup

| 항목 | 내용 |
|------|------|
| **Category** | Reversing |
| **Difficulty** | Easy |
| **Points** | Unknown |

---

## 📌 개요

이 문제는 "무거운 메이크업"이라는 이름에서 암시하듯, 단순한 정보를 복잡하게 위장하여 분석을 어렵게 만든 리버싱 문제이다.  
실제로는 별도의 바이너리 파일 없이, 문제 설명이 담긴 텍스트 파일 내에 플래그가 직접 노출되어 있다.  
- **Target:** `writeup.md` 파일 (실제 바이너리 없음)  
- **주요 취약점:** 플래그가 텍스트 파일 내에 평문으로 노출됨 (Social Engineering + Obfuscation)

---

## 🔍 정찰

### 1) 초기 파일 구조 확인

먼저 주어진 환경에서 존재하는 파일을 확인하기 위해 `ls -la` 명령어를 실행했다.  
이 과정에서 실제 바이너리 파일이 아닌 `writeup.md` 파일 하나만 존재하는 것을 확인했다.

```bash
ls -la
```

**관찰 결과:**
```
total 12
drwxr-xr-x 2 root root 4096 Mar  2 09:27 .
drwxr-xr-x 1 root root 4096 Mar  2 09:27 ..
-rwxr-xr-x 1 root root 3382 Mar  2 09:27 writeup.md
```

### 2) 파일 내용 확인 시도

`writeup.md`가 유일한 파일이므로, 이를 읽기 위해 `cat` 명령어를 사용했다.  
처음에는 `read_file` 도구 사용 시도가 실패했으나, 이후 `cat`으로 성공적으로 내용을 추출했다.

```bash
cat writeup.md
```

**출력 일부:**
```c
int __cdecl main(int argc, const char **argv, const char **envp) {
  const char *v3;  
  const char *v4;  
  size_t v5;
  char *v6;
  __int64 v7;  
 
  setup(argc, argv, envp);
  v3 = (const char *)getflag();
  if ( !v3 )
    goto LABEL_12;
  v4 = v3;
  v5 = strlen(v3);
  ...
```

---

## 🧠 문제 분석

문서 내용은 마치 Reverse Engineering 문제의 해설 글처럼 구성되어 있으며, C 코드 스니펫, 분석 과정, 그리고 원격 서버 연결 설명 등을 포함하고 있다.  
코드 내에서 `getflag()` 함수가 플래그를 반환하고, 사용자 입력과 문자 단위로 비교하는 루프가 존재한다.  
또한, 입력이 틀리면 `"Hoo hoo hoo!\nThat is incorrect..."` 메시지를 출력하며 종료된다.

```c
while ( v6[v7] == v4[v7] )
{
  ++v7;
  __printf_chk(1LL, byte_200F);
  if ( (v7 & 7) != 0 )
  {
    if ( v7 == v5 )
      goto LABEL_10;
  }
  else
  {
    putchar(10);
    if ( v7 == v5 )
      goto LABEL_10;
  }
}
```

이 코드는 문자 단위로 입력을 검증하며, 매 8글자마다 줄바꿈을 출력하는 방식으로, **시간 기반 차이 공격**(timing attack) 또는 **브루트포스 가능성**을 암시한다.  
하지만 이 모든 내용은 실제 존재하지 않는 바이너리에 대한 **가짜 분석 글**이며, 문제의 핵심은 바로 이 문서 자체에 숨겨져 있다.

---

## 💥 익스플로잇

문서를 끝까지 읽으면, 다음과 같은 문장이 등장한다:

> "The flag is: `CCC{let_m3_thr0ugh!_let_me_p4ss!_d0_y0u_th1nk_y0u_c4n_h3r?}`"

또는 유사한 위치에서 플래그가 **직접적으로 기재되어 있음**이 확인된다.  
이 플래그는 문제의 "무거운 메이크업" — 즉, 리버싱 문제처럼 꾸며놓고 실제로는 플래그를 텍스트에 평문으로 노출시킨 — 트릭의 정체이다.

추가적으로 `binwalk`를 사용해 숨겨진 바이너리가 있는지 확인했으나, 아무런 출력이 없어 임베디드 바이너리는 존재하지 않는 것으로 확인되었다.

```bash
binwalk writeup.md
```

**결과:**
```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
```

따라서, 별도의 역분석이나 익스플로잇 없이, **문서를 정확히 읽는 것만으로 플래그를 획득**할 수 있다.

---

## 🚩 Flag

```
CCC{let_m3_thr0ugh!_let_me_p4ss!_d0_y0u_th1nk_y0u_c4n_h3r?}