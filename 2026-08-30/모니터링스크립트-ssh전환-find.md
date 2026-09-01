# 2026-08-30 — 내가 만든 스크립트 디버깅, Tailscale SSH → 표준 sshd 전환, find

> 맥 미니 서버에서 세 가지를 했다.
> ① 직접 만든 모니터링 스크립트의 버그를 내가 찾아 고침
> ② Tailscale SSH 를 표준 sshd + 키 인증으로 바꿈 (이유를 대며 선택)
> ③ find 를 직접 조립하며 약점 보강
>
> **복붙이 아니라, 만들고 막히고 고친 날.**

---

# 1. 내가 만든 스크립트에 버그가 있었다

8/28에 vi 로 모니터링 스크립트 3개를 만들어뒀는데, 열어보니 **맥 명령어를 리눅스에 쓴** 버그가 있었다.

```
sys_monitor.sh    top -l 1 / PhysMem   ← macOS 전용
check_my_ip.sh    lsof (맥 형식)        ← 리눅스에선 파싱 안 맞음
check_users.sh    netstat (맥 형식)     ← 아직 안 고침
```

아마 AI 에게 물었는데 **맥 기준으로 답을 줬고**, 그대로 넣어서 안 맞았다. 주석에도 "macOS의 top", "macOS의 lsof" 라고 쓰여 있었다.

## 1.1 ⭐ 조용한 실패 — 로그를 열어보고 발견

`sys_monitor.sh` 는 에러 없이 잘 도는 것처럼 보였다. 그런데 로그를 열어보니:

```
[시각] |  (CPU 빔) |  (메모리 빔) | 디스크: 34G | 활성SSH: 0
       ↑ top -l 실패    ↑ PhysMem 없음
```

**절반이 빈칸이었다.** `top: 부적절한 옵션 -- 'l'` 이 나는데도 스크립트는 계속 돌며 빈칸을 기록했다.

> 8/5 백업 실패, 8/19 PDF 실패와 같은 유형 — **성공한 것처럼 보이는데 실패.**
> 로그(결과물)를 안 열어봤으면 "모니터링 잘 되네" 하고 넘어갔을 것이다.

## 1.2 고치기 — 리눅스 명령으로

명령을 직접 찾고, **스크립트에 넣기 전에 한 줄로 먼저 테스트**했다.

### 메모리: `top PhysMem` → `free`

```bash
free -h | grep Mem
# Mem:  7.4Gi  508Mi  6.8Gi  12Mi  243Mi  6.9Gi
#  $1    $2     $3     $4     $5     $6     $7
#       총계   사용   free  공유  버퍼   available
```

```bash
MEM_USAGE=$(free -h | grep Mem | awk '{print "메모리 사용: " $3 " / 여유: " $7}')
```

- `$3` = 사용, `$7` = available(실제 여유)
- **`$4`(free) 가 아니라 `$7`(available)** 을 쓴다 — 리눅스는 남는 메모리를 캐시로 쓰므로 `free` 는 실제보다 적게 나온다

### CPU: `top -l 1` → `top -bn1`

```bash
top -bn1 | grep '%Cpu'
# %Cpu(s):  0.0 us, 0.0 sy, 0.0 ni, 100.0 id, ...
#                                    $8=idle
```

```bash
CPU_USAGE=$(top -bn1 | grep '%Cpu' | awk '{print "CPU 사용률: " 100 - $8 "%"}')
```

- 리눅스 top 은 `-l` 대신 **`-bn1`** (배치모드, 1회)
- CPU 줄은 `%Cpu` (맥은 `CPU usage`)
- **`100 - idle` = 실제 사용률.** awk 가 계산도 해준다

> ⚠️ 남은 문제: `top -bn1` 은 1회 측정이라 첫 값이 부정확할 때가 있다(100% 로 튐).
> 정확히 하려면 `top -bn2` 로 두 번 재고 둘째 값을 쓰거나 `/proc/stat` 파싱. 나중에 개선.

## 1.3 겪은 오타 — 직접 발견

```
MEM_SAGE      → MEM_USAGE (U 빠짐) → 20행의 $MEM_USAGE 와 이름 안 맞아 빈칸
$7]           → $7}      (awk 는 중괄호 }) → 문법 오류
```

20행 REPORT 가 `$MEM_USAGE` 를 참조하는데 12행을 `MEM_SAGE` 로 저장 → **이름이 다르면 연결 안 됨.** `bash -n` 문법검사 + 실제 실행으로 잡았다.

---

# 2. ⭐ Tailscale SSH → 표준 sshd + 키 인증

## 2.1 발단 — Health check 경고

```
Health check: SELinux is enabled; Tailscale SSH may not work.
```

SSH 는 되고 있는데 경고가 떴다. **"되니까 괜찮겠지" 로 넘기지 않고** 파봤다.

## 2.2 지금 접속이 어느 경로인지 확인

```bash
pstree -ps $$
# systemd(1) ── tailscaled(1044) ── tailscaled ── bash
#                    ↑ sshd 가 아니라 tailscaled 가 내 셸의 부모
```

**Tailscale SSH 로 붙고 있었다.** 일반 sshd 를 안 거친다. 그래서 `who`/`last` 에 접속 기록이 안 남았던 것.

## 2.3 왜 표준 sshd 로 바꾸나 — SE 관점

| | Tailscale SSH | 표준 sshd + 키 |
|---|---|---|
| 편의 | 키 없이 바로 | 키 등록 필요 |
| 접속 기록 | **who/last 에 안 남음** | **pts/0 + IP 기록됨** |
| 표준성 | Tailscale 전용 기능 | 어디서나 통하는 표준 |
| SELinux | 충돌 경고 | 문제 없음 |

**목표(팀 서버 + 감사)에는 표준 sshd 가 맞다.** 접속 추적이 되고, 표준이라 이식 가능하고, SELinux 와 안 부딪힌다.

> 지난주에 배운 계층 분리 그대로:
> ```
> Tailscale    "안전한 길" (네트워크 터널)
> sshd + 키    "누가 들어오나" (인증) + 기록
> SELinux      "들어와서 뭘 하나" (권한)
> ```
> Tailscale SSH 는 이 셋을 Tailscale 이 다 삼킨 것 → 편하지만 계층이 뭉개진다.

## 2.4 전환 — "새 문 열리는 것 확인하고 옛 문 닫기"

원격에서 자기 접속을 끊어먹지 않게 순서가 중요하다.

```bash
# ① 맥북 공개키를 맥 미니에 등록 (키 인증 준비)
#    ssh-copy-id 는 Tailscale 재인증에 걸려 멈춤 → 직접 넣는 방식으로
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "ssh-ed25519 AAAA... john@Mac.local" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# ② 일반 sshd 로 붙는지 확인 (pstree 로 경로 검증)
#    → systemd ── sshd ── sshd-session ── bash  ✅

# ③ 그다음에 Tailscale SSH 만 끈다 (네트워크는 유지)
sudo tailscale set --ssh=false      # 맥 미니에서!

# ④ 경고 캐시 제거
sudo systemctl restart tailscaled
```

### 검증

```bash
who
# miniserver pts/0  15:25 (<TAILSCALE_IP>)   ← 접속 IP 가 기록됨!
tailscale debug prefs | grep RunSSH
# "RunSSH": false                            ← Tailscale SSH 꺼짐
```

**아까 안 나오던 접속 기록이 이제 who 에 남는다.** 원래 원하던 "누가 접속했나" 가 이걸로 해결됐다.

## 2.5 권한이 핵심

`.ssh` 는 700, `authorized_keys` 는 600 이어야 한다.
**sshd 는 권한이 헐거우면 키를 무시한다** (보안 장치). 이걸 안 맞추면 키를 등록해도 접속이 안 된다.

---

# 3. find 약점 보강

8/17 에 스스로 꼽은 약점(find/grep/awk 를 노트 없이 못 꺼냄). 직접 조립했다.

## 익힌 조합

```bash
find /etc -name "*.conf"              # 이름으로 (확장자엔 . 필요)
find /etc -name "*ssh*"               # 중간에 든 것 (앞뒤 * )
find /etc -mtime -7                    # 최근 7일 이내 수정
find /etc -name "*.conf" | wc -l       # 개수 세기 (파이프)
find /etc -mtime -7 2>/dev/null | grep ssh   # 에러 숨기고 거르기
find /etc -name "*ssh*" -exec ls -l {} \;    # 찾은 것에 명령 실행
```

## 배운 것

### `*` 의 위치가 의미를 바꾼다
```
"*ssh"     ssh 로 끝남      (sshd_config 안 걸림!)
"ssh*"     ssh 로 시작
"*ssh*"    아무 데나 ssh    ← 제일 넓음
```

### `-mtime` 의 부호
```
-7    최근 7일 이내       ← 이번
+7    7일보다 오래됨      (8/5 백업 정리 때 이거 씀)
7     정확히 7일 전
```

### "허가 거부" 는 정상일 수 있다
```
find: '/etc/sudoers.d': 허가 거부
find: '/etc/audit': 허가 거부
```
일반 사용자라 **민감한 폴더**(sudo 규칙, 감사 로그)엔 못 들어간다. 에러가 나도 나머지는 정상 검색됨.

### `2>/dev/null` 은 양날의 검
```
"허가 거부는 예상됨" 을 확인한 뒤 → 숨겨도 OK
"왜 안 되지" 디버깅 중          → 절대 숨기면 안 됨 (에러가 단서)
```
8/19 백업이 `2>/dev/null` 로 에러를 숨겨 조용히 실패한 것과 같은 교훈.

## ⭐ find 로 "최근 뭐가 바뀌었나" 보기

```bash
find /etc -mtime -7 2>/dev/null
```
결과에 오늘 내가 한 일이 그대로 보였다:
```
/etc/ssh/ssh_host_ed25519_key           ← SSH 작업
/etc/systemd/system/...tailscaled.service   ← Tailscale 설치
/etc/yum.repos.d/tailscale.repo          ← 저장소 추가
/etc/resolv.pre-tailscale-backup.conf    ← Tailscale DNS 백업
/etc/hostname                            ← macmini 로 변경
```
**"최근 수정 파일" = "내가 서버에 뭘 했나".** 장애 대응에서 "누가 뭘 건드렸지" 를 추적하는 방법.

---

# 4. 오늘 반복해서 나온 것

## "안 뜬다 ≠ 없다" (세 번)
```
grep acitve       → 오타. 검색어 잘못
ss: not found     → PATH 문제 (/usr/sbin 없음). 파일은 /usr/sbin/ss 에 있음
who 에 SSH 안 남음  → Tailscale SSH 라서. sshd 로 바꾸니 남음
```

## "엉뚱한 기계" (두 번)
```
authorized_keys 를 Mac 에 넣음 (Mac:~$ 프롬프트)
tailscale set 을 Mac 에서 실행
```
→ **프롬프트를 안 보고 명령을 쳤다.** 다만 이번엔 바로 알아챘다.
→ 환경 조치 필요: 맥 미니 프롬프트에도 색 구분 (rhel2 는 8/4 에 청록으로 함)

## "동작한다 ≠ 올바르다"
Tailscale SSH 는 **동작은 했지만** 접속 기록도 안 남고 SELinux 경고도 있었다.
표준 sshd 로 바꾼 건 "안 되던 걸 되게" 가 아니라 **"되는 걸 올바르게"** 한 것.

---

# 5. 면접에서 쓸 형태

```
❌ "Tailscale SSH 로 접속 구성했습니다"

✅ "처음엔 Tailscale SSH 로 편하게 붙였는데,
    접속 감사가 안 되고 SELinux 충돌 경고가 있어서
    Tailscale 은 네트워크 계층만 쓰고
    인증은 표준 sshd + 키로 분리했습니다"
```
**편한 기본값을 쓰다 한계를 발견하고, 이유를 대며 표준으로 바꾼 것** — 도구를 이해하고 선택한 사람의 말.

---

# 6. 남은 것

```
□ 재부팅 테스트 — 24시간 자동화 최종 검증 (SSH 방식 바꿨으니 유지되는지 확인)
□ 맥 미니 프롬프트 색 구분 (엉뚱한 기계 실수 방지)
□ check_users.sh 도 고치기 (netstat → ss)
□ sys_monitor CPU 100% 개선 (top -bn2)
□ awk 연습 (다음 약점)
□ <OBS_HOST> 서버 ZFS 공부 (find/grep 익혔으니 이제 가능)
```
