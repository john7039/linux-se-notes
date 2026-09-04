# 2026-09-04 — 관계형 DB 설계 + FastAPI 첫 실습

> 졸업프로젝트 백엔드를 대비해 **DB → 백엔드 → 인프라가 어떻게 이어지는지** 를
> 작은 예제로 직접 해봤다. "감이 안 온다" 던 것을 손으로 만져 잡은 날.

**환경**: rhel2 (실습용) · SQLite + FastAPI

---

## 1. 왜 했나

졸프에서 내 역할이 **백엔드 + DB + 인프라** 인데, 용어만으론 감이 안 왔다.
`users` 테이블 하나 + API 하나를 끝까지 만들어보며 흐름을 잡았다.

```
프론트(화면) ─ [API] ─ 백엔드(요청 처리) ─ DB(저장)
     상대            └──────── 나 ────────┘
```

---

## 2. DB — 테이블 만들고 넣고 꺼내기

### 테이블 생성 (CREATE)
```python
import sqlite3
conn = sqlite3.connect("posture.db")
conn.execute("""
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,   -- 각 사용자의 고유 열쇠
    email TEXT NOT NULL UNIQUE,             -- 필수 + 중복 불가
    name TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
)
""")
conn.commit(); conn.close()
```

### 넣기(INSERT)·꺼내기(SELECT)
```python
conn.execute("INSERT INTO users (email, name) VALUES (?, ?)", ("hong@test.com", "홍길동"))
for row in conn.execute("SELECT id, email, name FROM users"):
    print(row)
```

**결과**: `id`(1,2..)와 `created_at`(시각)이 **안 넣어도 자동으로** 채워짐 → AUTOINCREMENT·DEFAULT 가 설계대로 작동.

### ⭐ `?` 로 값 넣기 — 보안 습관
```
❌ f"... VALUES ('{email}')"     SQL 인젝션 위험
✅ "... VALUES (?)", (email,)    ? 로 분리 → 안전
```

---

## 3. 백엔드 — DB 작업을 "요청에 반응하게"

같은 DB 작업인데, `@app.get/post` 로 감싸면 **요청이 오면 실행**된다. 이게 백엔드.

```python
from fastapi import FastAPI
import sqlite3
app = FastAPI()

@app.get("/api/users")          # GET 요청 오면 → 꺼내기(SELECT)
def list_users():
    conn = sqlite3.connect("posture.db"); conn.row_factory = sqlite3.Row
    rows = conn.execute("SELECT id, email, name FROM users").fetchall()
    conn.close()
    return [dict(r) for r in rows]

@app.post("/api/users")         # POST 요청 오면 → 넣기(INSERT)
def add_user(email: str, name: str):
    conn = sqlite3.connect("posture.db")
    conn.execute("INSERT INTO users (email, name) VALUES (?, ?)", (email, name))
    conn.commit(); conn.close()
    return {"저장됨": True}
```

```
스크립트    내가 실행 → DB 조회
백엔드      브라우저가 요청 → API 가 DB 조회 → 응답
→ 같은 조회, "누가 시키냐" 가 다름
```

### 실행 (venv + uvicorn)
```bash
python3 -m venv .venv
.venv/bin/pip install fastapi uvicorn
.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
```

### ⭐ /docs 자동 생성
`http://<서버>:8000/docs` → API 문서가 그림으로 자동 생성 + 직접 테스트.
→ **이걸 팀원(프론트)에게 주면 API 규격 공유 끝.**

---

## 4. ⚠️ 겪은 문제 — 연결 거부 (인프라)

브라우저에서 접속하니 **"연결 거부".** 어제 배운 진단으로 좁혔다.

```
uvicorn 실행 중        ✅
0.0.0.0:8000 LISTEN   ✅ 서버는 잘 듣고 있음
그런데 밖에서 거부      → 방화벽!
```

원인: **firewalld 가 8000 포트를 안 열어둠** (RHEL 은 안 연 포트를 다 막음).

```bash
sudo firewall-cmd --add-port=8000/tcp    # 임시 (재부팅 시 사라짐)
```

→ 열고 나니 `/docs` 접속됨. `/docs` 에서 POST 하니 실제로 3번 사용자가 DB 에 저장됨.

### 이게 "배포의 현실"
```
로컬(localhost)     방화벽 상관없이 됨 → 개발자는 이걸 모름
실제 서버           방화벽·포트를 신경 써야 → 인프라 담당(나)의 영역
```

---

## 5. 오늘 이어본 전체 흐름

```
CREATE TABLE       데이터 담을 표 (DB)
   ↓
INSERT / SELECT    넣고 꺼내기 (SQL)
   ↓
@app.get/post      요청에 반응하게 감싸기 (백엔드)
   ↓
uvicorn 실행       서버 켜기
   ↓
방화벽 포트 열기     밖에서 접속되게 (인프라)
   ↓
/docs 로 확인       브라우저 → API → DB 저장 확인
```

**"프론트=화면 / 백엔드=DB조작 / 인프라=서버·방화벽" 을 말이 아니라 손으로 잡았다.**

---

## 6. 졸프와의 연결

```
오늘 (연습)    users 테이블 + API → rhel2
졸프 (실전)    posture_logs·calibrations·daily_reports + API → 맥 미니
→ 규모만 크지 흐름은 동일
```

---

## 남은 것

```
□ 관계형의 진짜 핵심 — 여러 테이블 + 외래키(user_id) 연결 (오늘은 단일 테이블)
□ UPDATE / DELETE (오늘은 INSERT/SELECT 만)
□ 맥 미니에 제대로 배포 (오늘은 rhel2 실습)
□ PostgreSQL (오늘은 SQLite)
```
