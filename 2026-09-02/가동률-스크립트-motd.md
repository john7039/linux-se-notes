# 2026-09-02 — 가동률 리포트 스크립트 + 접속 시 자동 표시(MOTD)

> 서버가 언제 켜지고 얼마나 가동됐는지 보여주는 스크립트를 직접 만들고,
> ssh 접속 시 자동으로 뜨도록(MOTD) 구성했다.
> **핵심: 데이터를 새로 쌓지 않고, 시스템이 이미 기록하는 `last`/`uptime` 을 읽어서 계산.**

---

# 1. 아이디어와 설계 판단

"서버 가동 시간을 기록하는 프로그램" 을 만들려다, **먼저 확인**했다.

```
직접 파일에 쌓기      →  용량 걱정. 그리고 중복
last / uptime        →  부팅·종료 이력을 시스템이 이미 기록 중 (/var/log/wtmp)
```

→ **읽어서 계산만 하면 파일을 안 쌓아도 된다.** 용량 문제 자체가 사라졌다.
(있는 것을 확인하고 시작한다 — 만들기 전에 `last reboot`, `uptime` 부터 봤다.)

---

# 2. 재료 (시스템이 주는 것)

```bash
uptime -s                          # 켜진 시각: 2026-09-02 01:34:48
uptime -p                          # 가동 시간: up 14 minutes
last reboot                        # 부팅 이력 전체
last reboot | grep -c "system boot"  # 부팅 횟수: 7
last reboot | grep "wtmp begins"     # 측정 시작 시점
```

---

# 3. 완성 스크립트 (`uptime_report.sh`)

```bash
#!/bin/bash
echo "===================================="
echo " 📊 맥 미니 서버 가동 현황"
echo "===================================="
echo "부팅 시각 : $(uptime -s)"
echo "가동 시간 : $(uptime -p)"
BOOT_COUNT=$(last reboot | grep -c "system boot")
echo "총 부팅   : ${BOOT_COUNT} 회"
echo "측정 시작 : $(last reboot | grep 'wtmp begins' | awk '{print $4, $5, $6}')"
echo "------------------------------------"
echo " 최근 부팅 이력 (최근 5회)"
echo "------------------------------------"
last reboot | grep "system boot" | head -5 | awk '{print "  " $5, $6, $7, $8}'
echo "===================================="
```

---

# 4. 배운 것 (오늘의 약점 = awk)

| 배운 것 | 내용 |
|---|---|
| `awk '{print $5,$6}'` | 지저분한 출력에서 **필요한 칸만** 뽑기 |
| `grep -c "패턴"` | 매칭 줄 개수 세기 (grep + wc 를 한 번에) |
| `$(명령)` | 명령 결과를 문자열·변수에 담기 |
| 칸 번호는 실제 출력으로 | `$3~$6` 로 넣었다가 빈칸 → 실제 줄 보고 `$4~$6` 로 수정 |

## ⭐ 위치(tail/head) 대신 내용(grep)으로 집기

처음엔 측정 시작 시점을 `last reboot | tail -2 | head -1` 로 집었는데 **빈 줄**이 걸려 빈칸이 나왔다.

```
tail -2 | head -1     위치로 집음 → last 끝의 빈 줄에 흔들림
grep "wtmp begins"    내용으로 집음 → 위치 상관없이 정확
```

→ **위치는 변수(빈 줄 등)에 흔들리고, 내용으로 집으면 안정적.** 실무에서 계속 쓰는 원칙.

---

# 5. MOTD — 접속 시 자동 표시

`~/.bashrc` 끝에 추가해, ssh 로 붙으면 위 리포트가 자동으로 뜨게 했다.

```bash
# 접속 시 서버 현황 표시 (MOTD)
if [[ $- == *i* ]]; then
    bash ~/uptime_report.sh
fi
```

## `$- == *i*` 조건이 핵심

```
붙이면    사람이 ssh 로 대화형 접속할 때만 뜸
안 붙이면  스크립트·자동화가 셸 열 때도 껴서 출력이 오염됨
```

- `.bashrc` 는 **대화형 로그인 셸**일 때 실행된다.
  `ssh host '명령'` 처럼 명령을 지정해 붙으면 `.bashrc` 를 읽지 않는다 → MOTD 안 뜸 (정상).
- 실무 서버가 접속 시 상태·정책을 띄우는 것과 같은 방식.

## ⚠️ .bashrc 편집은 안전망 두고

`.bashrc` 는 셸 시작 파일이라, 잘못 넣으면 접속 자체가 꼬일 수 있다.
→ **기존 세션은 닫지 말고, 새 창으로 테스트.** ("새 문 확인하고 옛 문 닫기")

---

# 6. 결과 (접속하면 자동으로)

```
📊 맥 미니 서버 가동 현황
부팅 시각 : 2026-09-02 01:34:48
가동 시간 : up 21 minutes
총 부팅   : 7 회
측정 시작 : Aug 24 05:18:17
 최근 부팅 이력 (최근 5회)
  Wed Sep 2 01:34
  Sun Aug 30 14:25
  ...
```

부팅 이력을 보면 **8/24 에 여러 번 재부팅** — 맥 미니 세팅하던 날의 흔적이 그대로 남아 있다.

---

# (오후 추가) logrotate — 로그가 무한정 안 쌓이게

> 5초마다 쌓이는 `sys_monitor.log` 를 어떻게 관리할지 고민하다,
> **"안 쌓기" 가 아니라 "쌓되 관리하기"** 가 답임을 배웠다. 그 표준 도구가 logrotate.

## 왜 필요한가

```
가동률 기록 (하루 몇 줄)      → 1년에 22KB. 무시 가능
sys_monitor.log (5초마다)    → 1년에 1.2GB.  ← 이게 진짜 용량 문제
```

logrotate = 로그가 일정 주기/크기를 넘으면 **오래된 건 압축·삭제**해서 자동 관리.
시스템이 이미 쓰고 있다 (`/etc/logrotate.d/` 에 dnf·firewalld·wtmp 등).

## 규칙 파일 문법

기존 예제(`/etc/logrotate.d/hawkey.log`)를 참고해 직접 작성.

```
/home/miniserver/system_monitor.log {
    daily          # 매일 회전 (weekly=매주)
    rotate 7       # 7개까지 보관 → 8일째부터 삭제 ★ 용량 관리의 핵심
    compress       # 오래된 건 .gz 압축 (텍스트는 90% 절감)
    missingok      # 파일 없어도 에러 안 냄
    notifempty     # 비어 있으면 회전 안 함
    create         # 회전 후 빈 로그 새로 생성 ★ 이걸 빠뜨렸다 고침
}
```

- **경로는 전체 경로**로 (`~` 는 logrotate 가 못 알아들음)

## 실행·검사 명령

```bash
logrotate -d 규칙파일                    # 검사만 (실제 회전 X, debug)
logrotate -f -s ~/logrotate.status 규칙파일   # 강제 회전
```

- `-s ~/logrotate.status` : 상태 파일을 홈에. 안 그러면 `/var/lib/logrotate/` 에
  쓰려다 **허가 거부** (일반 사용자라). → `-s` 로 회피

## 회전 결과 (눈으로 확인)

```
system_monitor.log        0바이트   ← create 가 만든 새 빈 파일
system_monitor.log.1.gz   방금 것    ← 원본이 압축돼 회전
system_monitor.log.2.gz   이전 것    ← 번호가 밀림 (7 넘으면 삭제)
```

`zcat system_monitor.log.1.gz` 로 풀어보니 원본 내용이 그대로 압축돼 있었다.

## ⭐ 겪은 함정 (이게 학습)

```
① 옵션 뒤에 한글 주석을 붙임
   daily          매일 회전    →  error: line 2 lines must begin with a keyword
   → 설정엔 옵션만. 주석은 # 로 별도 줄에

② create 를 빠뜨림
   회전은 됐는데(.gz 생김) 새 빈 로그가 안 생김
   → create 넣으니 해결. "없으면 어떻게 되는지" 를 직접 봄

③ 상태 파일 권한 거부
   /var/lib/logrotate/ 는 root 만 → -s 로 홈에 두어 회피
```

## 배운 원칙

```
"로그를 안 쌓는다" 가 아니라 "쌓되 자동으로 관리한다"
→ rotate N 개 · compress 로 용량 상한을 건다
→ 실무의 nginx·앱 로그가 다 이렇게 관리됨
```

---

# 남은 것

```
□ 가동률 % 계산 (제대로) — down 시간 합산, awk 산술 (다음 도전)
□ logrotate 를 /etc/logrotate.d/ 에 넣어 자동 실행 (지금은 수동 테스트만)
□ 재부팅 테스트 — UTM VM 자동 시작 (24시간 자동화 최종 조각)
□ 맥 미니 프롬프트 색 구분
```
