# 2026-07-29 — 1~2단계 복습: LVM · 권한 · SELinux

> 1~2단계를 전체적으로 훑고, 손에 안 익은 부분(LVM 전체 사이클, 권한 관리, SELinux)을 실습으로 다시 다짐.

---

## 0. 1~2단계 전체 지도

두 단계를 한 문장으로 구분하면:

> **1단계 = 서버 안쪽** (디스크 · 계정 · 프로세스)
> **2단계 = 서버와 바깥의 경계** (네트워크 · 접속 · 차단)

| 단계 | 영역 | 핵심 도구 |
|---|---|---|
| 1 | 디스크 & LVM | `pvs` `vgs` `lvs` `lvextend` `mkfs` `/etc/fstab` |
| 1 | 권한 & 계정 | `chmod` `chown` `groupadd` `usermod -aG` `id` |
| 1 | 프로세스 & 서비스 | `ps` `pgrep` `kill` `systemctl` |
| 2 | 네트워크 | `nmcli` |
| 2 | 접속 (SSH) | 키 인증, 포트 변경, `sshd_config` |
| 2 | 차단 (방화벽) | `firewall-cmd`, zone |
| 2 | 강제접근제어 | SELinux — 라벨, boolean |

### 이 조각들이 어떻게 이어지는가

어제 SSH 포트를 22 → <SSH_PORT>로 바꾼 작업이 1~2단계를 관통하는 예시다.

```
① sshd_config 편집     설정 파일 편집 (vi + 권한 이해 = 1단계)
② SELinux 포트 라벨    프로세스가 그 포트를 쓸 권한   (2단계)
③ firewalld 포트 허용  외부에서 그 포트로 들어올 권한 (2단계)
      ↓
  systemctl restart sshd    서비스 재시작 (1단계)
      ↓
  ss -tlnp 로 검증           상태 확인 (5단계 맛보기)
```

**리눅스 서버는 여러 층의 허가를 전부 통과해야 동작한다.** 어느 층에서 막혔는지 찾는 것이 트러블슈팅이다.

---

## 1. LVM — 전체 사이클 실습

### 3층 구조

| 층 | 뜻 | 명령 |
|---|---|---|
| **PV** (Physical Volume) | 물리 디스크·파티션을 LVM용으로 표시 | `pvcreate` `pvs` |
| **VG** (Volume Group) | PV들을 묶은 저장소 풀 | `vgcreate` `vgextend` `vgs` |
| **LV** (Logical Volume) | VG에서 잘라낸 논리 볼륨. 여기에 파일시스템을 얹음 | `lvcreate` `lvextend` `lvs` |

**왜 LVM인가** — 파티션은 크기를 못 바꾸지만 LV는 **서비스 중에도 확장**된다. 디스크가 모자라면 새 디스크를 VG에 추가하고 LV만 늘리면 된다.

### 조회 결과 읽기

```
$ sudo vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  rhel      1   3   0 wz--n- 61.41g      0     ← 꽉 참
  vg_data   1   1   0 wz--n- <10.00g <7.00g    ← 7G 여유
```

`rhel` VG가 `VFree 0` 인 상황이 실무에서 흔하다. "`/` 가 꽉 찼는데 늘릴 공간이 없다" → 새 디스크 추가 → `pvcreate` → `vgextend` → `lvextend` 순서로 해결.

```
$ sudo lvs
  lv_store vg_data -wi-ao----  3.00g
                    │   ││
                    │   │└─ o = open (마운트 중)
                    │   └─ a = active
                    └─ w = writeable
```

### ① 무중단 확장

```bash
sudo lvextend -r -L +2G /dev/vg_data/lv_store
```

| 옵션 | 뜻 |
|---|---|
| `-L +2G` | **2G 만큼 추가** (`-L 5G` 는 "5G로 맞춤" — `+` 유무가 완전히 다름) |
| `-r` | `--resizefs`. **파일시스템까지 같이 확장** |

**`-r` 이 핵심.** 빼면 LV(그릇)만 커지고 파일시스템(내용물)은 그대로라 `df` 상 용량이 안 늘어난다. 초보자 최다 실수.

→ 결과: `/store` 3G → **5G**, 마운트된 채로 확장됨

### ② 새 볼륨 만들기 — 전체 사이클

실무에서 "저장 공간 만들어주세요" 요청이 왔을 때 하는 작업 그대로.

```bash
sudo lvcreate -L 1G -n lv_backup vg_data       # ① LV 생성
sudo mkfs.xfs /dev/vg_data/lv_backup           # ② 파일시스템 포맷
sudo mkdir /backup                             # ③ 마운트 지점
sudo mount /dev/vg_data/lv_backup /backup      # ④ 임시 마운트
```

**LV는 빈 그릇일 뿐이다.** ②를 빼면 마운트가 실패한다.

### ③ fstab 등록 — 영구 마운트

`mount` 명령은 그 순간만 붙인다. **재부팅하면 사라진다.**

```bash
sudo cp /etc/fstab /etc/fstab.bak          # 백업 먼저 (습관)
sudo blkid /dev/vg_data/lv_backup          # UUID 확인
sudo vi /etc/fstab
```

추가할 줄:

```
UUID=<UUID>  /backup  xfs  defaults  0 0
```

| 칸 | 값 | 뜻 |
|---|---|---|
| 1 | `UUID=...` | 무엇을 |
| 2 | `/backup` | 어디에 |
| 3 | `xfs` | 파일시스템 종류 |
| 4 | `defaults` | 마운트 옵션 |
| 5 | `0` | dump 백업 여부 |
| 6 | `0` | 부팅 시 fsck 검사 순서 (`/`는 1, 나머지 2 또는 0) |

**왜 장치명이 아니라 UUID인가** — 디스크를 추가하면 `/dev/vdb` 가 `/dev/vdc` 로 밀릴 수 있다. UUID는 파일시스템에 새겨진 고유값이라 안 변한다. 실무 표준.

### ④ 검증 — 이걸 반드시 할 것

```bash
sudo umount /backup
sudo mount -a          # fstab에 적힌 것 전부 마운트 시도
df -hT /backup
sudo systemctl daemon-reload    # systemd에 fstab 변경 알림
```

> 🔥 **`mount -a` 검증 없이 재부팅했다가 부팅 불가에 빠지는 것이 리눅스 초보 사고 1위.** fstab 오타는 emergency mode로 직행한다.

추가 검증 도구:

```bash
findmnt --verify --verbose      # fstab 항목별로 UUID 해석·경로 존재를 검사
```

### 알아둘 것

- **XFS는 축소 불가.** 늘리는 건 되지만 줄일 수 없다 → LV는 **작게 시작해서 필요할 때 늘리는 게** 정석
- `df` 상 1G가 960M로 보이는 건 정상 (파일시스템 메타데이터·저널이 일부 사용)

---

## 2. 권한 & 소유권

### 문자열 읽는 법

```
d rwx r-x r-x    root   root
│ │   │   │       │      └─ 그룹
│ │   │   │       └─ 소유자
│ │   │   └─ others (나머지)
│ │   └─ group
│ └─ user (소유자)
└─ 종류 (d=디렉토리, -=일반파일, l=심볼릭링크)
```

```
r=4  w=2  x=1        rwx r-x r-x  →  755
```

**디렉토리에서의 의미가 파일과 다르다:**

| 권한 | 파일 | 디렉토리 |
|---|---|---|
| `r` | 내용 읽기 | **목록 조회** (`ls`) |
| `w` | 내용 수정 | **파일 생성·삭제** |
| `x` | 실행 | **진입** (`cd`) |

`--x` 조합이 유용하다 — `cd` 는 되지만 `ls` 는 안 된다. 정확한 파일명을 아는 사람만 접근시키고 싶을 때.

### 접근 권한 주는 3가지 방법

| 방법 | 명령 | 평가 |
|---|---|---|
| A. 소유자 변경 | `chown <LAB_USER>:<LAB_USER> /backup` | 간단하지만 여러 명이 쓸 땐 부적절 |
| B. 전부 열기 | `chmod 777` | **절대 금지.** "생각하기 귀찮아서 다 열었다"는 신호. 감점 요소 |
| C. **그룹 관리** | 아래 참조 | **실무 정석** |

### C. 그룹 + setgid (실무 방식)

```bash
sudo groupadd backupusers                 # 그룹 생성
sudo usermod -aG backupusers <LAB_USER>     # 사용자 추가
sudo chgrp backupusers /backup            # 디렉토리 그룹 변경
sudo chmod 2770 /backup                   # 그룹 rwx + setgid
```

결과: `drwxrws---. <LAB_USER> backupusers`

**`s` 가 setgid.** 그 안에 만들어지는 파일이 자동으로 `backupusers` 그룹을 상속받는다.

실습으로 확인한 차이:

```
group_test.txt   <LAB_USER>  backupusers   ← setgid 설정 후 생성
test.txt         <LAB_USER>  <LAB_USER>      ← 설정 전 생성
```

같은 사람이 같은 디렉토리에서 만들었는데 그룹이 다르다.

> ⚠️ **`usermod -aG` 의 `-a` 를 빼면 기존 그룹이 전부 날아간다.** `wheel` 에서 빠지면 sudo를 못 쓰게 된다. **`-aG` 를 한 덩어리로 외울 것.**

> ⚠️ **그룹 변경은 재로그인해야 적용된다.** 그룹 정보는 로그인 시 한 번 읽어 세션 내내 유지된다. `id` 로 확인.

### 특수 권한

| 값 | 이름 | 효과 |
|---|---|---|
| `4` | setuid | 소유자 권한으로 실행 (`passwd` 명령) |
| `2` | setgid | 하위 파일이 디렉토리 그룹을 상속 |
| `1` | sticky bit | **본인 파일만 삭제 가능** — `/tmp` 가 `1777` (`drwxrwxrwt`) |

### 왜 그룹 방식이 나은가

| | `chmod 777` | 그룹 + setgid |
|---|---|---|
| 접근 통제 | 불가 — 모든 계정 | 그룹 멤버만 |
| 사람 추가·제거 | 불가능 | `usermod -aG` 한 줄 |
| 파일 그룹 일관성 | 제각각 | 자동 통일 |
| 감사 대응 | "왜 다 열었나요?" | 설명 가능 |

---

## 3. SELinux

### 라벨(컨텍스트) 구조

**모든 것에 라벨이 붙는다** — 파일, 프로세스, 포트 전부.

```
system_u : object_r : sshd_config_t : s0
  사용자     역할        타입          보안수준
                         ↑
                 실무에서 볼 건 이것뿐
```

```bash
ls -Z 파일          # 파일 라벨
ps -eZ | grep 이름  # 프로세스 라벨
```

### 정책의 정체

SELinux 정책은 이런 문장들의 모음이다:

> `sshd_t` 프로세스는 `sshd_config_t` 파일을 읽을 수 있다
> `sshd_t` 프로세스는 `ssh_port_t` 포트에 바인딩할 수 있다
> `httpd_t` 프로세스는 `httpd_sys_content_t` 파일을 읽을 수 있다

어제 `semanage port -a -t ssh_port_t -p tcp <SSH_PORT>` 를 한 이유가 이것이다.

**일반 권한과는 완전히 별개의 층이다.**

```
접근 시도 → [일반 권한 rwx] → [SELinux 라벨] → 허용
```

### ⭐ mv vs cp — SELinux 이슈 1위

실습으로 확인:

```bash
echo "..." > /tmp/test.html
sudo mv /tmp/test.html /usr/share/nginx/html/     # → 403 Forbidden
sudo cp /tmp/test2.html /usr/share/nginx/html/    # → 200 OK
```

| 파일 | 라벨 | 결과 |
|---|---|---|
| `test.html` (mv) | `user_tmp_t` | **403** |
| `test2.html` (cp) | `httpd_sys_content_t` | **200** |

**일반 권한은 둘 다 `rw-r--r--` 로 동일했다.** 차이는 오직 라벨.

| 명령 | 라벨 처리 |
|---|---|
| `mv` | **원본 라벨을 그대로 가져옴** (파일을 통째로 이동) |
| `cp` | **목적지의 기본 라벨을 새로 받음** (내용만 복사) |

### 진단 — AVC 로그

```bash
sudo ausearch -m AVC -ts recent
```

```
avc: denied { read } for pid=... comm="nginx" name="test.html"
     scontext=system_u:system_r:httpd_t:s0          ← 주체 (누가)
     tcontext=unconfined_u:object_r:user_tmp_t:s0   ← 대상 (무엇을)
```

`httpd_t` 가 `user_tmp_t` 를 읽으려다 막혔다 — 원인이 한 줄에 다 있다.

더 친절한 버전 (해결책까지 제시):

```bash
sudo sealert -a /var/log/audit/audit.log
```

> 💡 **"뭔가 이상한데 로그에 아무것도 없다"** 싶으면 SELinux를 의심할 것. `ausearch -m AVC -ts recent` 는 습관적으로 쳐야 하는 명령.

### 해결 — 3단계 사고법

```
① 라벨이 틀렸나?       → restorecon 으로 복구
② 정책에 없는 경로인가? → semanage fcontext 로 규칙 추가 후 restorecon
③ 기능이 꺼져 있나?     → setsebool -P 로 켜기
```

**① restorecon** — "원래 있어야 할 라벨로 되돌려라"

```bash
sudo restorecon -v 파일
sudo restorecon -Rv /디렉토리/      # -R 재귀, -v 출력
```

**② semanage fcontext** — 기본 경로가 아닌 곳에 웹 파일을 둘 때

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"
sudo restorecon -Rv /data/web
```

`semanage fcontext` 가 **규칙을 등록**하고 `restorecon` 이 **그 규칙대로 적용**한다. 두 개가 한 세트.
(`chcon` 도 있지만 `restorecon` 하면 되돌아가므로 임시용)

**③ boolean** — 정책의 on/off 스위치

```bash
getsebool -a | grep httpd            # 목록
getsebool httpd_can_network_connect  # 현재 값
sudo setsebool -P httpd_can_network_connect on
```

| boolean | 뜻 |
|---|---|
| `httpd_can_network_connect` | 웹서버가 외부로 접속 (DB 연동, API 호출) |
| `httpd_can_network_connect_db` | DB 서버로만 접속 |
| `httpd_enable_homedirs` | 사용자 홈을 웹으로 서비스 |
| `httpd_use_nfs` | NFS 경로 읽기 |

기본값은 대부분 `off` — 최소 권한 원칙.

> ⚠️ **`-P` 를 빼면 재부팅 시 사라진다** (P = persistent)

### ⛔ 절대 하지 말 것

```bash
setenforce 0            # 임시 비활성화
SELINUX=disabled        # /etc/selinux/config 에서 영구 비활성화
```

인터넷에 제일 많이 나오는 답이지만, **면접에서 이렇게 답하면 탈락이다.** 문제를 해결한 게 아니라 보안 장치를 끈 것.

다만 **원인 파악용으로** 잠깐 `Permissive` 로 두는 건 실무에서 쓴다. `setenforce 0` 후 문제가 사라지면 "SELinux가 원인"이 확인된다. 확인 후 반드시 `setenforce 1` 로 되돌리고 제대로 고칠 것.

---

## 4. 실전 트러블슈팅 — httpd 포트 충돌

`sudo systemctl enable --now httpd` 가 실패했다.

### 진단 과정

```bash
systemctl status httpd --no-pager
# → (98)Address already in use: could not bind to address 0.0.0.0:80

httpd -t
# → Syntax OK          ← 설정 문제 아님

ss -tlnp | grep :80
# → 누군가 이미 80번을 쓰고 있음

systemctl is-active nginx
# → active             ← 범인
```

**nginx가 이미 80번을 잡고 있어서** 나중에 뜬 httpd가 실패한 것. 웹서버 두 개가 같은 포트를 쓸 수 없다.

### 교훈

**"서비스가 안 떠요"에서 멈추지 말 것.** `systemctl status` 와 `journalctl -xeu 서비스명` 에 원인이 거의 항상 적혀 있다.

```
① systemctl status        에러 메시지 확인
② 설정 문법 검사           httpd -t, nginx -t, sshd -T
③ ss -tlnp                포트 충돌 확인
④ ausearch -m AVC         SELinux 차단 확인
```

---

## 5. 오늘 발견한 관통 패턴 — 임시 vs 영구

**리눅스 전체가 이 구조다.**

| 영역 | 임시 (지금만) | 영구 (재부팅 후에도) |
|---|---|---|
| 마운트 | `mount` 명령 | `/etc/fstab` |
| 방화벽 | `firewall-cmd --add-port` | `--permanent` + `--reload` |
| 서비스 | `systemctl start` | `systemctl enable` |
| 네트워크 | `ip addr add` | `nmcli connection modify` |
| SELinux boolean | `setsebool` | `setsebool -P` |
| SELinux 라벨 | `chcon` | `semanage fcontext` + `restorecon` |

**새로운 걸 배울 때 "이건 임시야 영구야?"를 먼저 물으면 절반은 이해한 것이다.**

### 그리고 "바꾼 뒤 무엇을 다시 읽혀야 하는가"

| 바꾼 것 | 다시 읽히는 법 |
|---|---|
| hostname | 재접속 |
| 그룹 (`usermod -aG`) | 재로그인 |
| `/etc/fstab` | `systemctl daemon-reload` |
| firewalld permanent | `firewall-cmd --reload` |
| NetworkManager 설정 파일 | `nmcli connection reload` + `up` |
| sshd_config | `systemctl restart sshd` |

---

## 오늘의 서버 최종 상태

```
vdb 10G ─┬─ lv_store   5G  → /store    (3G에서 확장)
         └─ lv_backup  1G  → /backup   (새로 생성, fstab 등록, 재부팅 검증 완료)

/backup   drwxrws---  <LAB_USER>:backupusers   (setgid)
nginx     active (80번)     httpd  failed (포트 충돌 — disable 처리)
SELinux   Enforcing
```

---

## 복습 문제

<details>
<summary>1. <code>lvextend -L +2G</code> 와 <code>lvextend -L 2G</code> 의 차이는?</summary>

- `-L +2G` : 현재 크기에서 **2G 만큼 추가**
- `-L 2G` : 크기를 **2G로 맞춤** (지금이 5G라면 축소 시도 → XFS는 축소 불가라 실패)

`+` 하나 차이로 완전히 다른 명령이 된다.
</details>

<details>
<summary>2. <code>lvextend</code> 후 <code>df</code> 로 보니 용량이 안 늘었다. 왜?</summary>

`-r`(`--resizefs`) 옵션을 빼먹었다. LV(그릇)만 커지고 파일시스템(내용물)은 그대로다.

```bash
sudo lvextend -r -L +2G /dev/vg_data/lv_store    # 처음부터 -r
sudo xfs_growfs /store                            # 이미 늘렸다면 이걸로
```
</details>

<details>
<summary>3. fstab에 항목을 추가한 뒤 재부팅 전에 반드시 해야 할 검증은?</summary>

```bash
sudo umount /backup
sudo mount -a
```

`mount -a` 는 fstab의 모든 항목을 마운트 시도한다. **에러 없이 통과해야 안전**하다. 이 검증 없이 재부팅하면 오타 하나로 emergency mode에 빠진다.

추가로 `findmnt --verify --verbose` 로 항목별 검사도 가능.
</details>

<details>
<summary>4. 여러 명이 공유하는 디렉토리를 만들 때 <code>chmod 777</code> 대신 무엇을 쓰나?</summary>

**그룹 + setgid.**

```bash
sudo groupadd 그룹명
sudo usermod -aG 그룹명 사용자
sudo chgrp 그룹명 /디렉토리
sudo chmod 2770 /디렉토리
```

앞자리 `2`(setgid)로 하위 파일이 그룹을 자동 상속한다. 사람 추가·제거가 `usermod -aG` 한 줄로 되고, 그룹 밖은 접근 자체가 차단된다.
</details>

<details>
<summary>5. <code>/tmp</code> 에서 만든 파일을 웹 루트로 <code>mv</code> 했더니 403이 뜬다. 원인과 해결은?</summary>

**원인**: `mv` 는 원본 라벨(`user_tmp_t`)을 그대로 가져온다. 웹서버(`httpd_t`)는 `httpd_sys_content_t` 만 읽을 수 있어 거부된다. 일반 권한은 정상이라 눈치채기 어렵다.

**해결**:
```bash
sudo restorecon -v /usr/share/nginx/html/파일
```

**예방**: `mv` 대신 `cp` 를 쓰면 목적지 기본 라벨을 새로 받는다.
</details>

<details>
<summary>6. SELinux 문제를 만났을 때 해결 3단계는?</summary>

1. **라벨이 틀렸나** → `restorecon -Rv`
2. **정책에 없는 경로인가** → `semanage fcontext -a -t 타입 "경로(/.*)?"` 후 `restorecon`
3. **기능이 꺼져 있나** → `setsebool -P 이름 on`

진단은 `ausearch -m AVC -ts recent` 또는 `sealert -a /var/log/audit/audit.log`.

**`setenforce 0` 으로 끄는 건 해결이 아니다.**
</details>

<details>
<summary>7. 서비스가 시작되지 않을 때 확인 순서는?</summary>

```
① systemctl status 서비스명 --no-pager     에러 메시지
② journalctl -xeu 서비스명                  상세 로그
③ 설정 문법 검사 (httpd -t, nginx -t, sshd -T)
④ ss -tlnp                                포트 충돌
⑤ ausearch -m AVC -ts recent              SELinux 차단
```

오늘 httpd 실패는 ①에서 `Address already in use` 가 나왔고, ④로 nginx가 범인임을 확인했다.
</details>

<details>
<summary>8. 다음 설정들을 재부팅 후에도 유지하려면?  ① 마운트 ② 방화벽 포트 ③ 서비스 자동시작 ④ SELinux boolean</summary>

| | 임시 | 영구 |
|---|---|---|
| ① 마운트 | `mount` | `/etc/fstab` 등록 |
| ② 방화벽 | `firewall-cmd --add-port` | `--permanent` + `--reload` |
| ③ 서비스 | `systemctl start` | `systemctl enable` |
| ④ boolean | `setsebool` | `setsebool -P` |

전부 같은 구조다.
</details>
