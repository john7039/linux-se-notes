# 2026-07-27 — SSH 키 인증 & tmux

> 맥에서 RHEL VM으로 SSH 키 접속을 뚫고, tmux 기본기를 익힘.

---

## 1. SSH 키 인증

### 원리

키는 **한 쌍**으로 만들어진다.

```
~/.ssh/id_ed25519       개인키 (private)  — 절대 밖으로 나가면 안 됨. 내 맥에만 보관
~/.ssh/id_ed25519.pub   공개키 (public)   — 서버에 등록. 유출돼도 무방
```

접속할 서버의 `~/.ssh/authorized_keys` 에 **공개키를 한 줄 추가**하면 끝. 서버는 그 공개키로 "이 사람이 진짜 개인키를 가졌는지" 검증한다. 비밀번호가 오가지 않으므로 훔칠 수도, 무차별 대입할 수도 없다.

### 등록 방법 ①: ssh-copy-id (자동)

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub <LAB_USER>@<LAB_SERVER_IP>
```

**맥에서 실패했던 이유**: `~/.ssh/config` 에 `ForwardX11 yes` 가 걸려 있어서 SSH가 GUI 비밀번호 창(`ssh-askpass`)을 띄우려 했는데 맥엔 그게 없음 → 빈 비밀번호로 시도하다 실패.

해결:
```bash
DISPLAY= SSH_ASKPASS_REQUIRE=never ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host
```

### 등록 방법 ②: 수동 (실제로 이 방법으로 성공)

서버 콘솔에서:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... john@Mac.local' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
restorecon -Rv ~/.ssh          # ← RHEL 계열에서 필수
```

### ⚠️ 권한과 SELinux — 여기서 대부분 막힌다

| 대상 | 권한 | 이유 |
|---|---|---|
| `~/.ssh` | `700` | 남이 들여다보면 sshd가 키를 거부 |
| `~/.ssh/authorized_keys` | `600` | 위와 동일 |

sshd는 권한이 헐거우면 **에러도 안 내고 그냥 무시**한다. "분명 키 넣었는데 비밀번호를 또 묻는" 상황의 1순위 원인.

RHEL/CentOS에서는 여기에 **SELinux**가 하나 더 붙는다. 손으로 만든 파일은 라벨이 `ssh_home_t` 가 아니라서 sshd가 못 읽는다. `restorecon -Rv ~/.ssh` 로 라벨을 복구해야 한다.

```bash
ls -lZ ~/.ssh/authorized_keys     # -Z 옵션으로 SELinux 컨텍스트 확인
# unconfined_u:object_r:ssh_home_t:s0   ← 이게 정상
```

---

## 2. `~/.ssh/config` — 접속 정보 저장

매번 `ssh -i ~/.ssh/키 user@<LAB_SERVER_IP>` 치는 대신:

```
Host rhel
    HostName <LAB_SERVER_IP>
    User <LAB_USER>
    IdentityFile ~/.ssh/id_ed25519
    ForwardX11 no

Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ForwardX11 yes
```

이제 **`ssh rhel`** 한 줄로 접속. `scp`, `rsync`, `git` 도 이 별칭을 그대로 쓴다.

### ⚠️ 핵심 규칙: first-match-wins (먼저 매칭된 값이 이긴다)

같은 옵션이 여러 번 나오면 **처음 만난 값으로 확정**되고 뒤엣것은 **무시**된다.

```
Host *              ← 모든 호스트에 매칭됨
    ForwardX11 yes  ← 여기서 확정!

Host rhel
    ForwardX11 no   ← 무시됨. 이미 위에서 정해짐
```

그래서 **개별 호스트 블록을 항상 `Host *` 보다 위에** 둬야 한다.
"config 고쳤는데 왜 반영이 안 되지?"의 대부분이 이 규칙 때문.

---

## 3. tmux — 터미널 멀티플렉서

### 왜 쓰나

SSH가 끊겨도 서버에서 하던 작업이 **화면 그대로** 살아있다. 다시 붙으면(`attach`) 끊기기 직전 상태로 복귀.

|  | 백그라운드 실행 (`&`) | tmux |
|---|---|---|
| 프로세스 생존 | 상황에 따라 다름 (불확실) | **확실히 생존** |
| 화면 다시 보기 | **불가능** — 로그 파일로만 짐작 | `tmux attach` 로 그대로 복귀 |
| 대화형 작업 (`vim`, `top`, 설치 마법사) | 불가 | 가능 |

> 핵심은 "죽느냐"가 아니라 **"다시 붙을 수 있느냐"**.
> `dnf update` 돌리다 회선 끊기면 이 차이가 사고와 평온을 가른다.

### 계층 구조

```
세션(session)  ← 서버에 살아있는 작업 공간. 연결 끊겨도 안 죽음
 └ 윈도우(window)  ← 탭
    └ 페인(pane)   ← 화면을 쪼갠 조각
```

### 명령어 (셸에서)

```bash
tmux                      # 새 세션 시작 (이름 자동)
tmux new -s lab           # 'lab' 이름으로 세션 생성
tmux ls                   # 세션 목록
tmux attach -t lab        # 'lab' 세션에 붙기
tmux kill-session -t lab  # 세션 삭제
```

### 단축키 (prefix = `Ctrl-b`)

> `Ctrl-b` 를 누르고 **손을 뗀 다음** 다음 키를 누른다. 동시에 누르는 게 아님.

| 키 | 동작 |
|---|---|
| `Ctrl-b` `d` | **detach** — 세션 밖으로. 작업은 계속 돎 |
| `Ctrl-b` `%` | 좌우 분할 (세로선) |
| `Ctrl-b` `"` | 상하 분할 (가로선) |
| `Ctrl-b` `방향키` | 페인 이동 |
| `Ctrl-b` `c` | 새 윈도우 (탭) |
| `Ctrl-b` `숫자` | n번 윈도우로 이동 |
| `Ctrl-b` `[` | 스크롤 모드 (`q` 로 탈출) |
| `Ctrl-b` `x` | 현재 페인 닫기 |

암기 요령: `%` 는 사선이 **세로**로 서있다 → 세로 분할 / `"` 는 두 개가 **나란히** → 가로 분할

### 실무 배치

```
┌──────────────┬──────────────┐
│ tail -f      │  명령 치는 창 │
│ /var/log/    │  (설정 수정,  │
│ nginx/       │   서비스 재시작)│
│ error.log    ├──────────────┤
│              │  top          │
│ ← 에러 실시간 │  ← 리소스 감시│
└──────────────┴──────────────┘
```

설정 바꾸고 → 재시작하고 → **왼쪽 로그에 에러 뜨는지 즉시 확인**.

### 습관

> **SSH로 서버 들어가면 일단 `tmux` 부터 치고 시작한다.**

---

## 4. 실무 함정 2개 (실제로 겪음)

### ① `pkill -f` 가 자기 자신을 죽인다

```bash
ssh rhel 'pkill -f "tick"; echo "tick 프로세스 정리됨"'
# → SSH 연결이 exit 255 로 끊김
```

**원인**: `pkill -f` 는 **전체 명령줄**을 검사한다. 같은 줄의 `echo "tick 프로세스..."` 에도 `tick` 이 들어있으니, **자기가 실행 중인 셸을 매칭해서 죽여버린다.**

실무에서 서버 세션이 통째로 날아가는 대표 사고. 안전한 순서:

```bash
pgrep -af backup.sh      # ① 죽이기 전에 반드시 확인
kill 1234                # ② PID 지정해서 죽이기
pkill -f "[b]ackup.sh"   # 굳이 패턴을 쓸 거면 [] 트릭
```

`[b]ackup` 은 정규식으로 `backup` 에 매칭되지만, 명령줄에 적힌 문자열 자체는 `[b]ackup` 이라 자기 자신에는 매칭되지 않는다.

### ② 원격 백그라운드 실행이 SSH를 못 끊게 붙잡는다

```bash
ssh server './backup.sh &'      # ← ssh 가 끝나지 않고 멈춰있음
```

**원인**: 백그라운드 프로세스가 SSH 채널의 stdout/stderr 를 계속 잡고 있어서 연결을 닫지 못한다.

```bash
ssh server 'nohup ./backup.sh > /var/log/backup.log 2>&1 &'
#                             ^^^^^^^^^^^^^^^^^^^^^^^^^^ 출력을 끊어줘야 ssh 가 종료됨
```

---

## 5. 곁들여 나온 bash 조각

실습에 쓴 무한 루프:

```bash
i=0
while true; do
    i=$((i+1))
    echo "[$(date +%H:%M:%S)] 작업 진행중... $i초" | tee -a /tmp/job.log
    sleep 1
done
```

| 문법 | 뜻 |
|---|---|
| `i=0` | 변수 대입. **`=` 좌우에 공백 넣으면 에러** (bash 초보 1순위 실수) |
| `while true; do ... done` | 무한 반복. `true` 는 항상 성공하는 명령 |
| `$(( ))` | **산술 계산**. `i=$((i+1))` |
| `$( )` | **명령 실행 결과를 값으로 치환**. `$(date +%H:%M:%S)` → 현재 시각 |
| `sleep 1` | 1초 대기. 없으면 CPU 100% 점유 |

### `tee` — 화면과 파일에 동시 출력

```bash
./deploy.sh 2>&1 | tee -a /var/log/deploy.log
```

- `>` 만 쓰면 파일에만 들어가고 화면엔 안 보임
- 화면에만 찍으면 기록이 안 남음
- **둘 다** 원할 때 `tee`. `-a` 는 append(이어쓰기), 없으면 덮어씀
- `2>&1` 은 "표준에러(2)를 표준출력(1)이 가는 곳으로 합침" — 에러도 같이 기록하려면 필수

### `tail -f` — 로그 실시간 추적

```bash
tail -f /var/log/messages     # 파일이 커질 때마다 새 줄을 따라 출력
tail -F /var/log/messages     # 로그 로테이션으로 파일이 새로 생겨도 계속 추적 (실무는 -F 권장)
```

`Ctrl-c` 로 중단.

---

## 복습 문제

<details>
<summary>1. 서버에 공개키를 넣었는데도 비밀번호를 묻는다. 확인할 것 3가지는?</summary>

1. `~/.ssh` 권한이 `700`, `authorized_keys` 가 `600` 인지 (`ls -ld ~/.ssh`)
2. SELinux 컨텍스트가 `ssh_home_t` 인지 (`ls -lZ`) → 아니면 `restorecon -Rv ~/.ssh`
3. 소유자가 본인 계정인지 (root 로 만들었다면 `chown`)

추가로 `sudo journalctl -u sshd` 또는 `ssh -v` 로 로그를 보면 원인이 바로 나온다.
</details>

<details>
<summary>2. ssh config 에서 <code>Host *</code> 아래에 <code>Host rhel</code> 블록을 뒀는데 설정이 안 먹는다. 왜?</summary>

SSH config 는 **first-match-wins**. `Host *` 가 모든 호스트에 먼저 매칭되어 값이 확정되므로, 뒤에 오는 `Host rhel` 의 같은 옵션은 무시된다. **개별 호스트 블록을 위로 올려야** 한다.
</details>

<details>
<summary>3. tmux 에서 세션을 유지한 채 빠져나오는 키는? 그리고 다시 들어가는 명령은?</summary>

- 나가기: `Ctrl-b` 그다음 `d` (detach)
- 들어가기: `tmux attach -t 세션이름` (목록 확인은 `tmux ls`)
</details>

<details>
<summary>4. <code>pkill -f "nginx"</code> 를 실행했더니 SSH 세션이 끊겼다. 원인과 안전한 대안은?</summary>

`pkill -f` 가 전체 명령줄을 검사하는데, 자신이 실행 중인 셸의 명령줄에도 `nginx` 문자열이 포함되어 자기 자신을 죽였다.

대안: `pgrep -af nginx` 로 먼저 확인 → `kill <PID>` 로 PID 지정. 또는 `pkill -f "[n]ginx"`.
</details>

<details>
<summary>5. <code>./job.sh > log 2>&1 &</code> 에서 <code>2>&1</code> 의 의미는?</summary>

표준에러(fd 2)를 표준출력(fd 1)이 향하는 곳으로 합친다는 뜻. 즉 에러 메시지도 `log` 파일에 함께 기록된다.

**순서 주의**: `2>&1 > log` 는 의도대로 동작하지 않는다. 리다이렉션은 왼쪽부터 처리되므로 `> log` 를 먼저 써야 한다.
</details>
