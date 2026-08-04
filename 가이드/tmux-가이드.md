# tmux 가이드

> 원격 서버 작업의 필수 도구. **SSH가 끊겨도 작업이 살아남는다.**
> 명령어 사전이 아니라 **"이럴 때 이렇게"** 중심의 레퍼런스.

---

# 왜 쓰나

```
SSH 접속 → 작업 시작 → 회선 끊김 → 작업 통째로 날아감
SSH 접속 → tmux 안에서 작업 → 회선 끊김 → 다시 붙으면 그대로 있음
```

| | 백그라운드 실행 (`&`) | tmux |
|---|---|---|
| 프로세스 생존 | 상황에 따라 다름 | **확실히 생존** |
| 화면 다시 보기 | **불가능** — 로그로만 짐작 | `attach` 로 그대로 복귀 |
| 대화형 작업 (`vim`, `top`, 설치 마법사) | 불가 | 가능 |

> 핵심은 "죽느냐"가 아니라 **"다시 붙을 수 있느냐"**.
> `dnf update` 돌리다 회선이 끊기면 이 차이가 사고와 평온을 가른다.

## ⚠️ 한계 — 재부팅은 못 버틴다

| 실행 방식 | 로그아웃 | 재부팅 |
|---|---|---|
| 셸에서 그냥 실행 | 죽음 | 죽음 |
| `&` 백그라운드 | 상황에 따라 | 죽음 |
| **tmux 안에서** | **살아있음** | **죽음** |
| **systemd 서비스** | 살아있음 | **살아있음** |

**tmux는 "작업 세션"용이지 "상시 서비스"용이 아니다.** 재부팅 후에도 돌아야 하는 것은 systemd에 등록한다.

---

# 3계층 구조

```
세션(session)     서버에 살아있는 작업 공간. 연결이 끊겨도 안 죽음
 └ 윈도우(window)   탭
    └ 페인(pane)     화면을 쪼갠 조각
```

**세션 = 프로젝트, 윈도우 = 작업 종류, 페인 = 동시에 봐야 하는 것.** 이렇게 나눠 쓰면 정리가 된다.

---

# 셸에서 쓰는 명령

```bash
tmux                          # 새 세션 (이름 자동)
tmux new -s <이름>             # 이름 지정해서 생성
tmux ls                       # 세션 목록
tmux attach -t <이름>          # 세션에 붙기  (줄여서 tmux a -t <이름>)
tmux attach                   # 마지막 세션에 붙기
tmux kill-session -t <이름>    # 세션 삭제
tmux kill-server              # 전부 종료
```

**서버에서 이름 없는 세션이 쌓이면 헷갈린다.** 항상 `-s` 로 이름을 주는 습관을 들일 것.

```bash
tmux new -s deploy            # 배포 작업용
tmux new -s monitor           # 모니터링용
```

---

# 단축키 (prefix = `Ctrl-b`)

> `Ctrl-b` 를 누르고 **손을 뗀 다음** 다음 키를 누른다. 동시에 누르는 게 아니다.

## 필수 — 이것만 알면 시작 가능

| 키 | 동작 |
|---|---|
| `Ctrl-b` `d` | **detach** — 세션 밖으로. 작업은 계속 돎 |
| `Ctrl-b` `%` | 좌우 분할 (세로선) |
| `Ctrl-b` `"` | 상하 분할 (가로선) |
| `Ctrl-b` `방향키` | 페인 이동 |
| `Ctrl-b` `c` | 새 윈도우 (탭) |
| `Ctrl-b` `숫자` | n번 윈도우로 |
| `Ctrl-b` `[` | 스크롤 모드 (`q` 로 탈출) |

**암기 요령**: `%` 는 사선이 **세로**로 서있다 → 세로 분할 / `"` 는 두 개가 **나란히** → 가로 분할

## 페인 조작

| 키 | 동작 |
|---|---|
| `Ctrl-b` `x` | 현재 페인 닫기 (확인 후) |
| `Ctrl-b` `z` | **현재 페인만 전체화면 토글** ← 매우 유용 |
| `Ctrl-b` `o` | 다음 페인으로 순환 |
| `Ctrl-b` `q` | 페인 번호 표시 (숫자 누르면 이동) |
| `Ctrl-b` `{` / `}` | 페인 위치 바꾸기 |
| `Ctrl-b` `Ctrl-방향키` | 페인 크기 조절 |
| `Ctrl-b` `공백` | 레이아웃 순환 |

> **`Ctrl-b` `z` 를 기억할 것.** 분할해놓고 한 페인을 자세히 봐야 할 때 잠깐 키웠다가 다시 돌린다.

## 윈도우 조작

| 키 | 동작 |
|---|---|
| `Ctrl-b` `c` | 새 윈도우 |
| `Ctrl-b` `,` | 윈도우 이름 변경 |
| `Ctrl-b` `n` / `p` | 다음 / 이전 윈도우 |
| `Ctrl-b` `w` | 윈도우 목록에서 고르기 |
| `Ctrl-b` `&` | 윈도우 닫기 |

## 세션 조작

| 키 | 동작 |
|---|---|
| `Ctrl-b` `d` | detach |
| `Ctrl-b` `s` | 세션 목록에서 전환 |
| `Ctrl-b` `$` | 세션 이름 변경 |

## 기타

| 키 | 동작 |
|---|---|
| `Ctrl-b` `?` | **단축키 전체 목록** ← 막히면 이것 |
| `Ctrl-b` `:` | 명령 모드 (vi 의 `:` 같은 것) |
| `Ctrl-b` `t` | 시계 표시 |

---

# 스크롤 · 복사 모드

터미널에서 마우스 휠이 안 먹거나, 지나간 출력을 봐야 할 때.

```
Ctrl-b [          복사 모드 진입
  방향키 / PgUp    이동
  q                탈출
```

## 검색

복사 모드 안에서:

| 키 | 동작 |
|---|---|
| `/` | 아래 방향 검색 |
| `?` | 위 방향 검색 |
| `n` / `N` | 다음 / 이전 결과 |
| `g` / `G` | 맨 위 / 맨 아래 |

> **긴 로그 출력에서 에러를 찾을 때** 유용하다. `Ctrl-b [` → `?error` → Enter.

## 텍스트 복사

기본(emacs) 키 기준:

```
Ctrl-b [          복사 모드
  공백             선택 시작
  방향키           범위 선택
  Enter            복사 (tmux 버퍼에 저장)
Ctrl-b ]          붙여넣기
```

vi 키 모드로 바꿨다면 선택은 `v`, 복사는 `y` (아래 설정 참조).

```bash
tmux show-buffer              # 버퍼 내용 확인
tmux save-buffer ~/out.txt    # 파일로 저장
```

> ⚠️ **tmux 버퍼는 맥 클립보드와 별개다.** 맥으로 가져가려면 마우스 드래그로 선택하거나(마우스 모드 켰을 때), `tmux save-buffer` 로 파일에 저장한 뒤 옮긴다.

---

# 실무 활용 패턴

## ① 로그 보면서 작업하기

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

설정 바꾸고 → 재시작하고 → **왼쪽 로그에 에러 뜨는지 즉시 확인.** 창을 오갈 필요가 없다.

## ② 장시간 작업

```bash
tmux new -s update
sudo dnf update -y
# Ctrl-b d 로 나가서 다른 일 하다가
tmux attach -t update
```

**회선이 끊겨도 업데이트는 계속 진행된다.**

## ③ 서버 여러 대 동시 관리

윈도우 하나에 서버 하나씩:

```bash
tmux new -s ops
# Ctrl-b c → ssh rhel   → Ctrl-b , → 이름을 "lab" 으로
# Ctrl-b c → ssh rhel2  → Ctrl-b , → 이름을 "db" 로
```

`Ctrl-b w` 로 목록을 보며 오갈 수 있고, **윈도우 이름이 곧 서버 이름**이라 헷갈리지 않는다.

## ④ 습관

> **SSH로 서버에 들어가면 일단 `tmux` 부터 친다.**

나중에 회선이 끊겨봐야 이 습관의 가치를 안다.

---

# 설정 파일 `~/.tmux.conf`

서버마다 만들어두면 편하다. 아래는 실용적인 최소 설정.

```bash
# ── prefix 를 Ctrl-a 로 (Ctrl-b 는 손이 멀다)
set -g prefix C-a
unbind C-b
bind C-a send-prefix          # Ctrl-a 를 앱에 그대로 전달하려면 두 번

# ── 마우스 (스크롤·클릭 선택·크기조절)
set -g mouse on

# ── 윈도우 번호를 1부터 (0은 손이 멀다)
set -g base-index 1
setw -g pane-base-index 1

# ── 스크롤백 길이
set -g history-limit 10000

# ── 복사 모드를 vi 키로
setw -g mode-keys vi

# ── 분할을 직관적인 키로 (| 와 -)
bind | split-window -h
bind - split-window -v

# ── 설정 리로드
bind r source-file ~/.tmux.conf \; display "reloaded"

# ── 상태바
set -g status-position bottom
set -g status-interval 5
set -g status-right "#H | %Y-%m-%d %H:%M"
```

**적용**

```bash
tmux source-file ~/.tmux.conf     # 실행 중인 세션에 적용
# 또는 위 설정을 넣었다면  Ctrl-a r
```

## ⚠️ 중첩 tmux 주의

로컬에서 tmux를 쓰면서 서버에서도 tmux를 쓰면 **prefix가 겹친다.**

```
로컬 prefix   Ctrl-a
서버 prefix   Ctrl-b     ← 다르게 설정하면 충돌 없음
```

또는 안쪽 tmux에 명령을 보내려면 **prefix를 두 번** 누른다 (`Ctrl-a` `Ctrl-a` `d`).

---

# 문제 해결

| 증상 | 원인 · 조치 |
|---|---|
| `no server running on /tmp/tmux-1000/default` | 세션이 하나도 없음. **서버를 재부팅하면 tmux 세션은 사라진다** (정상) |
| `sessions should be nested with care` | 이미 tmux 안에서 `tmux` 를 또 실행함. `exit` 후 다시 |
| 화면이 깨져 보임 | `Ctrl-b` `:` → `refresh-client -S`, 또는 터미널 크기 조정 |
| 마우스 스크롤이 안 됨 | `set -g mouse on` 설정 필요 |
| detach 했는데 세션이 안 보임 | 다른 사용자로 만든 세션. `whoami` 확인 |
| 색이 이상함 | `tmux -2` 로 실행하거나 `set -g default-terminal "screen-256color"` |

---

# 빠른 참조

```
tmux new -s <이름>        생성
tmux ls                   목록
tmux a -t <이름>          붙기
tmux kill-session -t <이름>  삭제

Ctrl-b d      나가기 (작업 유지)
Ctrl-b %      좌우 분할
Ctrl-b "      상하 분할
Ctrl-b 방향키  페인 이동
Ctrl-b z      페인 전체화면 토글
Ctrl-b c      새 윈도우
Ctrl-b w      윈도우 목록
Ctrl-b [      스크롤/복사 모드 (q 로 탈출)
Ctrl-b ?      단축키 전체 목록
```

---

## 관련 문서

- [서버 운영 절차](서버-운영-절차.md)
- [vi 가이드](vi-가이드.md)
