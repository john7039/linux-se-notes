# 2026-07-28 — vi · 네트워크 설정 · firewalld · SSH 보안

> vi 기본기를 익히고, 네트워크 설정을 읽는 법을 배운 뒤, firewalld와 SSH 포트 변경까지 실제로 적용함.

---

## 1. vi 편집기

### 모드 — 이것부터 이해해야 한다

vi는 다른 편집기와 근본적으로 다르다. **글자를 쳐도 입력되지 않는다.**

```
일반 모드 (Normal)   ← 시작하면 여기. 키보드가 전부 "명령"
입력 모드 (Insert)   ← 여기서만 글자가 써짐
```

| 키 | 동작 |
|---|---|
| `i` | 입력 모드 진입 |
| `Esc` | 일반 모드 복귀 |

**막히면 무조건 `Esc`.** 화면 아래 `-- INSERT --` 표시로 현재 모드를 알 수 있다.

### 저장 · 종료

| 명령 | 뜻 |
|---|---|
| `:w` | 저장 (write) |
| `:q` | 나가기 (quit) |
| `:wq` | 저장하고 나가기 |
| `:q!` | **저장 안 하고 강제 종료** |

`:q!` 가 탈출구다. 설정 파일 만지다 이상해지면 이걸로 빠져나오면 아무 일도 없던 게 된다.

### 이동

| 키 | 동작 |
|---|---|
| `gg` | 파일 맨 처음 |
| `G` | 파일 맨 끝 |
| `숫자G` | n번째 줄로 점프 |
| `0` | 줄 맨 앞 |
| `$` | 줄 맨 뒤 |

`숫자G` 가 특히 중요하다. `nginx.conf line 47: syntax error` → `vi nginx.conf` → `47G` → 바로 그 줄.

### 검색 · 치환

| 명령 | 동작 |
|---|---|
| `/단어` | 아래 방향 검색 |
| `n` / `N` | 다음 / 이전 결과 |
| `:%s/찾을것/바꿀것/g` | 전체 치환 (`%`=파일전체, `s`=substitute, `g`=줄 내 전부) |

### 편집

| 키 | 동작 |
|---|---|
| `dd` | 줄 삭제 |
| `yy` | 줄 복사 |
| `p` | 붙여넣기 |
| `x` | 글자 하나 삭제 — **주석 `#` 뗄 때 제일 많이 씀** |
| `O` | 위에 새 줄 삽입 + 입력 모드 |
| `u` | 되돌리기 (undo) |
| `Ctrl-r` | 다시 실행 (redo) |

숫자를 앞에 붙이면 반복된다: `3dd`(3줄 삭제), `5yy`(5줄 복사)

**`u` 가 생명줄이다.** 여러 번 눌러 계속 거슬러 올라갈 수 있다.

### 설정

```
:set nu        줄 번호 켜기 (설정 파일 다룰 땐 필수)
:set nonu      끄기
```

---

## 2. 오늘 겪은 삽질 3개 ← 제일 값진 부분

### ① 맥과 서버를 혼동했다

맥 터미널에서 `vi ~/practice.conf` 를 열었는데 계속 빈 화면이었다. 파일은 **VM 안에** 있었고, 맥에는 없었다.

```
[맥]  ────ssh────>  [VM: RHEL]
 여기서 vi 열면      여기에 파일이 있음
 빈 파일
```

**`ssh rhel` 로 접속한 뒤에** 작업해야 한다. 지금 어디인지 확인하는 법:

```bash
hostname     # Mac → 맥 / rhel-lab → 서버
```

> 서버 여러 대를 오가면 엉뚱한 서버에 명령을 치는 사고가 난다. 그래서 서버마다 hostname을 다르게 잡아 프롬프트에 표시되게 한다.

### ② vi는 없는 파일을 조용히 새로 만든다

경로를 잘못 쳐도 **에러 없이 빈 파일이 열린다.** 그래서 오타를 눈치채기 어렵다.

**파일을 열었는데 비어 있으면 → 경로 오타를 의심할 것.** `:set nu` 로 줄 번호를 켜보면 내용이 있는지 바로 안다.

### ③ 설정 파일은 대부분 root 전용이다

```bash
$ cp /etc/ssh/sshd_config ~/practice.conf
cp: cannot open '/etc/ssh/sshd_config' for reading: 허가 거부

$ ls -l /etc/ssh/sshd_config
-rw------- 1 root root 3674 ...      ← 600 = root만 읽고 쓸 수 있음
```

| 파일 | 권한 | 비고 |
|---|---|---|
| `/etc/ssh/sshd_config` | `600` | 서버 정책 — 일반 사용자에게 안 보임 |
| `/etc/ssh/ssh_config` | `644` | 클라이언트 설정 — 누구나 읽기 가능 |
| `*.nmconnection` | `600` | 네트워크 설정 |

**명령이 실패하면 다음 명령을 치기 전에 에러 메시지를 읽자.** ①과 ③이 겹쳐서 한참 헤맸다.

---

## 3. hostname 설정

```bash
hostnamectl                                  # 현재 상태
sudo hostnamectl set-hostname rhel-lab       # 설정
exit && ssh rhel                             # 재접속해야 프롬프트에 반영됨
```

`localhost` → `rhel-lab` 으로 변경. 프롬프트가 `[<LAB_USER>@rhel-lab ~]$` 가 된다.

> **`ssh rhel` 의 `rhel` 은 서버 이름이 아니다.** 맥의 `~/.ssh/config` 에 적어둔 **별명**일 뿐이고, 서버 본인의 이름(hostname)과는 별개다.

---

## 4. 네트워크 설정 (nmcli)

### device vs connection

- **device** = 랜카드 자체 (`enp0s1`) — 하드웨어 관점
- **connection** = 그 카드에 적용할 **설정 프로필** — 설정 관점

한 장치에 프로필을 여러 개 두고 바꿔 끼울 수 있다.

### 조회

```bash
nmcli device status                    # 장치 상태
nmcli connection show                  # 프로필 목록
nmcli connection show enp0s1           # 프로필 상세
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns connection show enp0s1
```

`-f` 로 원하는 필드만 골라 본다.

### 현재 설정 (이미 고정 IP였음)

```
ipv4.method:      manual              ← DHCP 아님, 고정
ipv4.addresses:   <LAB_SERVER_IP>/24
ipv4.gateway:     <GATEWAY_IP>
ipv4.dns:         8.8.8.8, 1.1.1.1
```

### 설정 파일 위치

```
/etc/NetworkManager/system-connections/enp0s1.nmconnection    (권한 600)
```

**RHEL 9부터 이 형식(keyfile)** 으로 바뀌었다. RHEL 8까지는 `/etc/sysconfig/network-scripts/ifcfg-eth0` 였다. 오래된 교재는 `ifcfg-` 로 설명하니 헷갈리지 말 것.

### ⚠️ 파일을 직접 고치면 바로 적용되지 않는다

NetworkManager가 메모리에 든 설정을 계속 쓴다.

```bash
sudo nmcli connection reload      # 파일 다시 읽기
sudo nmcli connection up enp0s1   # 적용
```

**권장은 파일을 직접 안 고치는 것.** `nmcli` 로 바꾸면 저장과 적용을 알아서 해준다.

```bash
sudo nmcli connection modify enp0s1 ipv4.dns "8.8.8.8 1.1.1.1"
sudo nmcli connection up enp0s1
```

---

## 5. firewalld

### 방화벽은 원래부터 켜져 있다

리눅스 커널에 **netfilter** 라는 패킷 검사 엔진이 내장돼 있다. `firewalld` 는 그걸 조작하는 **관리 도구**다. 방화벽을 "세우는" 게 아니라 이미 있는 방화벽에 **문을 열어주는** 것.

```
외부 → [netfilter (커널)] → 서비스
            ↑ firewalld가 규칙을 설정
```

| 방향 | 기본 정책 |
|---|---|
| **들어오는 연결** (inbound) | **차단** — 허용한 것만 통과 |
| 나가는 연결 (outbound) | 허용 |

그래서 `dnf update` 나 `curl` 은 설정 없이도 되고, 밖에서 들어오는 것만 열어줘야 한다.

### 세대 구분 (면접 단골)

| 이름 | 역할 |
|---|---|
| **netfilter** | 커널 내부의 실제 엔진 |
| **nftables** | 규칙 문법 체계. RHEL 8부터 기본 |
| **iptables** | 옛 방식. 지금은 nftables로 변환되어 동작 |
| **firewalld** | 위를 쉽게 다루는 관리 계층 (zone 제공) |

```bash
sudo nft list ruleset      # firewalld가 만든 실제 규칙 확인
```

### zone

신뢰 수준별로 규칙을 묶어둔 것.

| zone | 신뢰도 | 용도 |
|---|---|---|
| `drop` | 최저 | 전부 차단, 응답도 안 함 |
| `public` | 낮음 | **기본값.** 명시적으로 연 것만 허용 |
| `home` / `work` | 중간 | 내부망 |
| `trusted` | 최고 | 전부 허용 |

### ⚠️ 최대 함정 — runtime vs permanent

**firewalld는 규칙을 두 벌 들고 있다.**

```
runtime    지금 동작 중. --reload / 재부팅에 사라짐
permanent  파일에 저장. 재부팅해도 유지
```

**`--reload` = permanent 파일을 읽어서 runtime을 그 내용으로 교체**

- permanent에 **없는** runtime 규칙 → 사라짐
- permanent에 **있는** 규칙 → 유지 (오히려 복원됨)

> 실습 중 `--reload` 해도 8080이 안 사라져서 당황했는데, 앞서 실수로 `--permanent` 로 추가해둔 게 있어서였다. **정상 동작이다.**

**실무 패턴 — 이 두 줄이 한 세트**

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

### 명령

| 명령 | 용도 |
|---|---|
| `--list-all` | 현재 zone 전체 현황 |
| `--list-ports` / `--list-services` | 포트만 / 서비스만 |
| `--permanent --list-all` | **저장된** 설정 확인 |
| `--add-port=8080/tcp` | 포트 열기 |
| `--add-service=http` | 서비스 이름으로 열기 (**권장**) |
| `--remove-port=` / `--remove-service=` | 닫기 |
| `--reload` | permanent 적용 |
| `--add-port=8080/tcp --timeout=5m` | 5분 뒤 자동 닫힘 — **자기 자신을 잠그는 사고 방지** |

**포트 번호보다 서비스 이름을 쓸 것.** 의도가 명확하고, 여러 포트를 쓰는 서비스도 한 번에 처리된다.

### 포트를 열어도 서비스가 없으면 소용없다

| 상황 | 결과 |
|---|---|
| 방화벽 열림 + 서비스 실행 중 | 접속 성공 |
| 방화벽 열림 + 서비스 없음 | **connection refused** |
| 방화벽 닫힘 + 서비스 실행 중 | **timeout** (응답 없음) |

**refused와 timeout의 차이로 원인을 구분**할 수 있다. 실무 트러블슈팅의 기본 감별법.

---

## 6. SSH 포트 변경 — 세 군데를 손봐야 한다

오늘의 핵심. **하나라도 빠지면 안 된다.**

```
① sshd_config 의 Port
② SELinux 포트 라벨
③ firewalld 포트 허용
```

### 안전 수칙

**작업 중인 SSH 세션을 절대 닫지 말 것.** 설정을 잘못해도 기존 세션이 살아있으면 되돌릴 수 있다. 테스트는 **새 터미널**에서 한다. (sshd를 재시작해도 이미 연결된 세션은 안 끊긴다)

### 0단계 — 백업

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

설정 파일 건드리기 전 백업은 습관으로.

### ① sshd_config 편집

```bash
sudo vi /etc/ssh/sshd_config
```

`#Port 22` → `Port <SSH_PORT>` 로 변경 (`#` 제거 필수)

### ② SELinux 포트 등록 ← RHEL의 핵심 관문

**이걸 빼먹으면 sshd가 시작조차 안 된다.**

```bash
sudo semanage port -l | grep ssh              # ssh_port_t  tcp  22
sudo semanage port -a -t ssh_port_t -p tcp <SSH_PORT>
sudo semanage port -l | grep ssh              # ssh_port_t  tcp  <SSH_PORT>, 22
```

SELinux는 sshd가 아무 포트나 쓰게 두지 않는다. **`ssh_port_t` 라벨이 붙은 포트에만** 바인딩할 수 있다.

> **방화벽 vs SELinux**
> - 방화벽: **누가 들어올 수 있는가**
> - SELinux: **각 프로그램이 무엇을 해도 되는가**
>
> sshd가 해킹당해도 SELinux가 허용한 범위 밖으론 아무것도 못 한다.

### ③ 방화벽 허용

```bash
sudo firewall-cmd --permanent --add-port=<SSH_PORT>/tcp
sudo firewall-cmd --reload
```

### 재시작 및 확인

```bash
sudo systemctl restart sshd
systemctl status sshd --no-pager        # active (running)
ss -tlnp | grep ssh                     # 0.0.0.0:<SSH_PORT>
```

새 터미널에서:

```bash
ssh -p <SSH_PORT> <LAB_USER>@<LAB_SERVER_IP>
```

### 맥의 `~/.ssh/config` 갱신

```
Host rhel
    HostName <LAB_SERVER_IP>
    Port <SSH_PORT>          ← 추가
    User <LAB_USER>
    IdentityFile ~/.ssh/id_ed25519
    ForwardX11 no
```

### 참고 — socket activation

RHEL 일부 환경에서 `sshd.socket` 이 활성이면 `sshd_config` 의 Port가 **무시된다.** 확인:

```bash
systemctl is-active sshd.socket      # inactive 여야 정상
systemctl is-active sshd.service     # active
```

---

## 7. 비밀번호 인증 끄기

포트를 옮긴 건 자동 스캔을 줄이는 정도지 진짜 방어가 아니다. **비밀번호 인증을 끄는 것**이 훨씬 효과적이다 — 무차별 대입이 원천 봉쇄된다.

```
PasswordAuthentication no
```

### ⚠️ Include 주의

RHEL 9부터 `sshd_config` 맨 위에 이 줄이 있다:

```
Include /etc/ssh/sshd_config.d/*.conf
```

**SSH는 first-match-wins**(먼저 읽은 값이 이김)이므로, `sshd_config.d/` 안의 파일이 본 파일을 이긴다. "분명 바꿨는데 안 먹힌다"의 원인.

**최종 적용값 확인:**

```bash
sudo sshd -T | grep -i passwordauthentication
```

`sshd -T` 는 모든 설정 파일을 읽어 **실제로 적용되는 값**을 출력한다.

### 진짜로 적용됐는지 확인하는 법

`sshd -T` 는 *파일에 뭐라고 적혀 있는지*만 보여준다. 재시작을 안 했다면 실제 동작은 다를 수 있다. **직접 시도해보는 게 확실하다:**

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password user@host
```

```
적용 전:  Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password)
적용 후:  Permission denied (publickey,gssapi-keyex,gssapi-with-mic)
                                                     ↑ password 사라짐
```

거부 메시지의 **괄호 안 목록이 서버가 허용하는 인증 방식**이다.

---

## 오늘의 서버 최종 상태

| 항목 | 값 |
|---|---|
| hostname | `rhel-lab` |
| IP | `<LAB_SERVER_IP>/24` (고정) |
| SSH 포트 | **<SSH_PORT>** |
| 비밀번호 인증 | **비활성** (키 전용) |
| 방화벽 | `cockpit dhcpv6-client http https ssh` + `<SSH_PORT>/tcp` |
| SELinux | Enforcing, `ssh_port_t` 에 <SSH_PORT> 등록 |
| 맥 접속 | `ssh rhel` |

---

## 복습 문제

<details>
<summary>1. vi에서 파일을 열었는데 화면이 비어 있다. 무엇을 의심해야 하나?</summary>

**경로 오타.** vi는 존재하지 않는 파일을 열면 에러 없이 **빈 새 파일**을 만든다. `:q!` 로 나와서 `ls` 로 파일이 실제로 있는지, 지금 접속한 곳이 맥인지 서버인지(`hostname`) 확인한다.
</details>

<details>
<summary>2. SSH 포트를 22에서 <SSH_PORT>로 바꾸려면 몇 군데를 고쳐야 하나?</summary>

**세 군데.**
1. `/etc/ssh/sshd_config` 의 `Port <SSH_PORT>`
2. SELinux — `semanage port -a -t ssh_port_t -p tcp <SSH_PORT>`
3. firewalld — `firewall-cmd --permanent --add-port=<SSH_PORT>/tcp` + `--reload`

②를 빼먹으면 sshd가 아예 시작되지 않는다.
</details>

<details>
<summary>3. <code>firewall-cmd --add-port=8080/tcp</code> 로 포트를 열었는데 재부팅하니 닫혔다. 왜?</summary>

`--permanent` 없이 추가하면 **runtime에만** 적용된다. 재부팅이나 `--reload` 시 permanent 설정으로 덮어써지면서 사라진다.

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```
</details>

<details>
<summary>4. 서비스에 접속했더니 "connection refused"가 떴다. 방화벽 문제일까?</summary>

**아닐 가능성이 높다.** 방화벽이 막으면 보통 **timeout**(무응답)이 난다. refused는 포트까지 도달했지만 **거기서 듣고 있는 서비스가 없다**는 뜻이다.

`ss -tlnp` 로 서비스가 실제로 떠 있는지 먼저 확인한다.
</details>

<details>
<summary>5. <code>sshd_config</code> 에서 <code>PasswordAuthentication no</code> 로 바꿨는데 여전히 비밀번호로 로그인된다. 원인 두 가지는?</summary>

1. **sshd를 재시작하지 않았다** — `sudo systemctl restart sshd`
2. **`/etc/ssh/sshd_config.d/*.conf` 가 덮어쓰고 있다** — `Include` 가 맨 위에 있고 SSH는 first-match-wins이므로 그쪽이 이긴다.

최종 적용값은 `sudo sshd -T | grep -i passwordauthentication` 으로 확인한다.
</details>

<details>
<summary>6. 방화벽과 SELinux의 역할 차이는?</summary>

- **방화벽(firewalld)**: 네트워크 관점 — **누가 들어올 수 있는가**
- **SELinux**: 프로세스 관점 — **각 프로그램이 무엇을 해도 되는가**

SSH 포트를 바꿀 때 둘 다 손봐야 하는 이유가 이것이다. 방화벽은 <SSH_PORT>로 들어오는 걸 허용하고, SELinux는 sshd가 <SSH_PORT>에 바인딩하는 걸 허용한다.
</details>
