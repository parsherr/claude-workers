> **Bağlantılar:** [[CLAUDE]] | [[agent-generator/anatomy-team-lead]] | [[agent-generator/anatomy-worker]] | [[agent-generator/rules]] | [[agent-generator/team-patterns]] | [[researches/003-claude-code-headless-spawn-mekanizmasi]] | [[researches/004-worker-manager-ipc-iletisim]] | [[researches/005-task-queue-persistence]]

# Sistem Tasarımı: Team Lead + Worker Altyapısı

*Spawn mekanizması, IPC, state machine, hata yönetimi ve tüm bileşenlerin bütünleşik mimarisi.*

**Kaynak araştırmalar:** Research 003, 004, 005, 006 | Tarih: 2026-08-21

---

## 1. Büyük Resim

```
┌─────────────────────────────────────────────────────────┐
│                     KULLANICI                           │
│              "Şu görevi yap"                            │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   TEAM LEAD AGENT                       │
│           (claude -p, model: opus)                      │
│                                                         │
│  1. Görevi analiz et                                    │
│  2. Multi-agent gerekli mi? (Princeton NLP testi)       │
│  3. Alt görevlere böl                                   │
│  4. Worker'ları spawn et                                │
│  5. Sonuçları topla, sentezle                           │
│  6. Kalite gate uygula                                  │
│  7. Raporla                                             │
└──────────────────────┬──────────────────────────────────┘
                       │
           ┌───────────┼───────────┐
           │           │           │
           ▼           ▼           ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │Worker-1 │  │Worker-2 │  │Worker-3 │
    │(analyst)│  │(builder)│  │(reviewer│
    │ haiku   │  │ sonnet  │  │ sonnet  │
    └────┬────┘  └────┬────┘  └────┬────┘
         │             │             │
    result.json    result.json   result.json
         │             │             │
         └─────────────┴─────────────┘
                       │
                  SQLite tasks.db
                  (state tracking)
```

---

## 2. Spawn Mekanizması (Research 003)

### 2.1 Team Lead Spawn

Team Lead, kullanıcı tarafından veya ana Claude Code session'ından başlatılır:

```bash
claude -p "$(cat task-brief.txt)" \
  --permission-mode dontAsk \
  --allowedTools "Task,Read,Glob,Grep,TodoWrite" \
  --output-format stream-json \
  --max-turns 30 \
  --model opus \
  --no-session-persistence
```

**Kritik:** `--permission-mode dontAsk` — `--dangerously-skip-permissions` değil. (Research 003: headless ortamda blocking dialog açar, asılır.)

### 2.2 Worker Spawn (Team Lead tarafından)

Team Lead her worker için Task tool'u kullanır — veya Manager Script subprocess olarak başlatır:

**Task tool ile (Claude Code native):**
```
Task tool çağrısı içinde worker brief'i gönder.
Worker kendi context window'unda çalışır.
Sonuç Task tool'un return value'su olarak gelir.
```

**Manager Script subprocess olarak (üretim):**

```bash
#!/usr/bin/env bash
# worker-spawn.sh

TASK_ID="$1"
TASK_CONTENT="$2"
WORKER_TYPE="$3"   # analyst | builder | reviewer | executor | researcher

WORKER_DIR="/tmp/claude-worker-${TASK_ID}"
mkdir -p "${WORKER_DIR}/.claude"

# Her worker izole config dizini alır (Research 003: ~/.claude/ paylaşımı session bozar)
CLAUDE_CONFIG_DIR="${WORKER_DIR}/.claude" \
timeout 600 \
claude -p "${TASK_CONTENT}" \
  --permission-mode dontAsk \
  --allowedTools "$(worker_tools ${WORKER_TYPE})" \
  --output-format json \
  --max-turns 20 \
  --model "$(worker_model ${WORKER_TYPE})" \
  --no-session-persistence \
  > "${WORKER_DIR}/result.json" 2> "${WORKER_DIR}/stderr.log"

EXIT_CODE=$?

# Silent failure check (Research 003 kritik bulgusu)
DENIALS=$(jq '.permission_denials | length // 0' "${WORKER_DIR}/result.json" 2>/dev/null || echo "99")
if [ "$DENIALS" -gt 0 ] || [ "$EXIT_CODE" -ne 0 ]; then
  echo "WORKER_FAILED: task_id=${TASK_ID}, exit=${EXIT_CODE}, denials=${DENIALS}"
  exit 1
fi

echo "WORKER_DONE: task_id=${TASK_ID}"

# Tool/model seçimi fonksiyonları
worker_tools() {
  case "$1" in
    analyst)    echo "Read,Glob,Grep,WebSearch" ;;
    builder)    echo "Read,Glob,Grep,Write,Edit,Bash" ;;
    reviewer)   echo "Read,Glob,Grep,Bash" ;;
    executor)   echo "Bash,Read,Write" ;;
    researcher) echo "WebSearch,WebFetch,Read,Write" ;;
    *)          echo "Read,Glob,Grep" ;;
  esac
}

worker_model() {
  case "$1" in
    analyst)    echo "haiku" ;;
    executor)   echo "sonnet" ;;
    *)          echo "sonnet" ;;
  esac
}
```

---

## 3. IPC Mimarisi (Research 004)

**Seçilen hibrit:** SQLite task queue + file-based result

```
Manager/TeamLead                     Worker
      │                                │
      │  1. INSERT task (pending)       │
      │  ──────────────────────────►    │
      │                                │
      │  2. spawn subprocess ──────────►
      │                           [brief'i okur]
      │                           [çalışır]
      │                           [result.json yazar]
      │                           [exit 0/1]
      │◄── process exit event ─────────│
      │                                │
      │  3. result.json okur            │
      │  4. SQLite UPDATE (done/failed) │
      │  5. /tmp/worker-{id}/ temizle  │
```

### 3.1 Görev Payload Formatı

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "code_review",
  "worker_type": "reviewer",
  "objective": "Review auth.py for security vulnerabilities",
  "scope": {
    "in": ["src/auth.py", "tests/test_auth.py"],
    "out": ["everything else"]
  },
  "inputs": {
    "cwd": "/project",
    "files": ["/project/src/auth.py"]
  },
  "completion_criteria": [
    "All functions in auth.py reviewed",
    "Security issues with confidence >= 80 reported",
    "Output format matches specification"
  ],
  "constraints": {
    "max_turns": 20,
    "allowed_tools": ["Read", "Grep", "Glob"],
    "budget_usd": 0.50
  },
  "output_format": "STATUS / CONFIDENCE / SUMMARY / ARTIFACTS / ISSUES"
}
```

### 3.2 Result Payload Formatı

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "done",
  "confidence": 92,
  "result": "Reviewed auth.py. Found 2 critical issues...",
  "artifacts": [],
  "issues": "SQL injection risk in login() at line 47",
  "permission_denials": [],
  "total_cost_usd": 0.023,
  "session_id": "claude-session-uuid",
  "exit_code": 0,
  "turns_used": 8,
  "completed_at": "2026-08-21T14:30:00Z"
}
```

---

## 4. State Machine (Research 005)

```
                  ┌──────────┐
                  │  PENDING │ ← Task oluşturuldu
                  └────┬─────┘
                       │ Worker claim eder (atomic)
                  ┌────▼─────┐
                  │ RUNNING  │ ← Worker aktif
                  └────┬─────┘
           ┌───────────┴────────────┐
           │                        │
      ┌────▼─────┐            ┌─────▼────┐
      │   DONE   │            │  FAILED  │
      └──────────┘            └─────┬────┘
                                    │ retry_count < max_retries
                               ┌────▼─────┐
                               │  PENDING │ ← Yeniden kuyruğa
                               └──────────┘

Ek durum: STALE (RUNNING ama worker süreci ölmüş)
  → Stale recovery: updated_at + 30s timeout → PENDING'e geri al
```

### SQLite Şeması (Research 005'ten)

```sql
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;

CREATE TABLE IF NOT EXISTS tasks (
    id              TEXT PRIMARY KEY,
    type            TEXT NOT NULL,
    worker_type     TEXT NOT NULL DEFAULT 'builder',
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK(status IN ('pending','running','done','failed')),
    priority        INTEGER NOT NULL DEFAULT 0,
    payload         TEXT NOT NULL,      -- JSON görev detayı
    result          TEXT,               -- JSON worker sonucu
    confidence      INTEGER,            -- 0-100 worker confidence
    error           TEXT,
    worker_id       TEXT,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 3,
    claim_id        INTEGER NOT NULL DEFAULT 0,  -- Stale claim tespiti
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT NOT NULL DEFAULT (datetime('now')),
    started_at      TEXT,
    completed_at    TEXT,
    timeout_seconds INTEGER NOT NULL DEFAULT 600
);

CREATE INDEX idx_tasks_status_priority ON tasks(status, priority DESC, created_at);
CREATE INDEX idx_tasks_stale ON tasks(status, updated_at) WHERE status = 'running';
```

### Atomic Claim (Race Condition Önlemi)

```sql
-- Birden fazla Manager aynı anda çalışırsa race condition önlenir
BEGIN IMMEDIATE;
SELECT id, payload FROM tasks
  WHERE status = 'pending'
  ORDER BY priority DESC, created_at ASC
  LIMIT 1;
-- Sonuç varsa:
UPDATE tasks
  SET status = 'running',
      worker_id = :worker_id,
      claim_id = claim_id + 1,
      started_at = datetime('now'),
      updated_at = datetime('now')
  WHERE id = :task_id AND status = 'pending';
COMMIT;
```

---

## 5. Manager Script — Ana Döngü

```python
#!/usr/bin/env python3
"""
Manager Script: Team Lead + Worker altyapısının orkestratörü.
Team Lead'in Task tool'u yerine kullanılır — daha fazla kontrol için.
"""

import sqlite3, subprocess, json, os, time, uuid, signal
from pathlib import Path

DB_PATH = Path.home() / ".claude-workers" / "tasks.db"
MAX_WORKERS = 5
POLL_INTERVAL = 1  # saniye
STALE_TIMEOUT = 30  # saniye

active_workers: dict[str, subprocess.Popen] = {}

def claim_task(db_path: Path, worker_id: str) -> dict | None:
    """Atomic claim: pending → running"""
    with sqlite3.connect(db_path) as conn:
        conn.row_factory = sqlite3.Row
        conn.execute("PRAGMA busy_timeout=5000")
        conn.execute("BEGIN IMMEDIATE")
        row = conn.execute(
            "SELECT id, payload, worker_type FROM tasks "
            "WHERE status='pending' ORDER BY priority DESC, created_at ASC LIMIT 1"
        ).fetchone()
        if not row:
            conn.execute("ROLLBACK")
            return None
        conn.execute(
            "UPDATE tasks SET status='running', worker_id=?, claim_id=claim_id+1, "
            "started_at=datetime('now'), updated_at=datetime('now') WHERE id=?",
            (worker_id, row['id'])
        )
        conn.execute("COMMIT")
        return dict(row)

def spawn_worker(task: dict) -> subprocess.Popen:
    """Worker subprocess başlat"""
    task_id = task['id']
    worker_dir = Path(f"/tmp/claude-worker-{task_id}")
    worker_dir.mkdir(parents=True, exist_ok=True)
    (worker_dir / ".claude").mkdir(exist_ok=True)

    payload = json.loads(task['payload'])
    # Brief'i dosyaya yaz — çok uzun prompt'u inline geçme
    brief_file = worker_dir / "brief.txt"
    brief_file.write_text(format_worker_brief(payload))

    env = os.environ.copy()
    env['CLAUDE_CONFIG_DIR'] = str(worker_dir / ".claude")

    tool_map = {
        'analyst':    'Read,Glob,Grep,WebSearch',
        'builder':    'Read,Glob,Grep,Write,Edit,Bash',
        'reviewer':   'Read,Glob,Grep,Bash',
        'executor':   'Bash,Read,Write',
        'researcher': 'WebSearch,WebFetch,Read,Write',
    }
    model_map = {
        'analyst': 'haiku',
    }

    worker_type = task.get('worker_type', 'builder')
    tools = tool_map.get(worker_type, 'Read,Glob,Grep')
    model = model_map.get(worker_type, 'sonnet')

    result_file = worker_dir / "result.json"

    proc = subprocess.Popen(
        [
            'claude', '-p', brief_file.read_text(),
            '--permission-mode', 'dontAsk',
            '--allowedTools', tools,
            '--output-format', 'json',
            '--max-turns', '20',
            '--model', model,
            '--no-session-persistence',
        ],
        stdout=open(result_file, 'w'),
        stderr=open(worker_dir / "stderr.log", 'w'),
        env=env,
    )
    proc._worker_dir = worker_dir  # type: ignore
    proc._task_id = task_id        # type: ignore
    return proc

def collect_result(task_id: str, proc: subprocess.Popen, db_path: Path):
    """Worker tamamlandı — sonucu oku ve DB'yi güncelle"""
    worker_dir = proc._worker_dir  # type: ignore
    result_file = worker_dir / "result.json"
    exit_code = proc.returncode

    status = 'failed'
    confidence = 0
    result_text = None
    error = None

    if result_file.exists():
        try:
            raw = json.loads(result_file.read_text())
            # Silent failure check (Research 003)
            denials = len(raw.get('permission_denials', []))
            if denials > 0 or exit_code != 0:
                error = f"exit_code={exit_code}, denials={denials}"
            else:
                # Worker'ın kendi STATUS satırını parse et
                result_text = raw.get('result', '')
                status, confidence = parse_worker_output(result_text)
        except (json.JSONDecodeError, KeyError) as e:
            error = f"result parse failed: {e}"
    else:
        error = "no result file — worker may have crashed"

    with sqlite3.connect(db_path) as conn:
        conn.execute(
            "UPDATE tasks SET status=?, result=?, confidence=?, error=?, "
            "completed_at=datetime('now'), updated_at=datetime('now') WHERE id=?",
            (status, result_text, confidence, error, task_id)
        )

    # Temizlik
    import shutil
    shutil.rmtree(worker_dir, ignore_errors=True)

def recover_stale_workers(db_path: Path):
    """Süreci ölmüş ama running görünen task'ları kurtarır"""
    with sqlite3.connect(db_path) as conn:
        stale = conn.execute(
            "SELECT id FROM tasks WHERE status='running' "
            f"AND updated_at < datetime('now', '-{STALE_TIMEOUT} seconds')"
        ).fetchall()
        for (task_id,) in stale:
            if task_id not in active_workers:  # Gerçekten stale
                conn.execute(
                    "UPDATE tasks SET status='pending', worker_id=NULL, "
                    "updated_at=datetime('now') WHERE id=? AND retry_count < max_retries",
                    (task_id,)
                )
                conn.execute(
                    "UPDATE tasks SET status='failed', error='stale_no_retry', "
                    "updated_at=datetime('now') WHERE id=? AND retry_count >= max_retries",
                    (task_id,)
                )

def main_loop(db_path: Path):
    manager_id = f"manager-{os.getpid()}"
    print(f"Manager started: {manager_id}")

    while True:
        # 1. Tamamlanan worker'ları topla
        done = [(tid, p) for tid, p in active_workers.items() if p.poll() is not None]
        for task_id, proc in done:
            collect_result(task_id, proc, db_path)
            del active_workers[task_id]

        # 2. Stale worker recovery
        recover_stale_workers(db_path)

        # 3. Yeni task al (kapasite varsa)
        while len(active_workers) < MAX_WORKERS:
            task = claim_task(db_path, manager_id)
            if not task:
                break
            proc = spawn_worker(task)
            active_workers[task['id']] = proc
            print(f"Spawned worker for task: {task['id']}")

        time.sleep(POLL_INTERVAL)

def parse_worker_output(text: str) -> tuple[str, int]:
    """Worker output'undan STATUS ve CONFIDENCE çıkar"""
    status = 'failed'
    confidence = 0
    for line in (text or '').splitlines():
        if line.startswith('STATUS:'):
            val = line.split(':', 1)[1].strip().lower()
            if val == 'done':
                status = 'done'
            elif val == 'partial':
                status = 'failed'  # partial = retry candidate
        elif line.startswith('CONFIDENCE:'):
            try:
                confidence = int(line.split(':', 1)[1].strip())
            except ValueError:
                pass
    return status, confidence

def format_worker_brief(payload: dict) -> str:
    """Payload'dan worker brief string'i oluştur"""
    return f"""<worker-brief>
<task-id>{payload.get('task_id', 'unknown')}</task-id>
<objective>{payload.get('objective', '')}</objective>
<scope>
  <in>{json.dumps(payload.get('scope', {}).get('in', []))}</in>
  <out>{json.dumps(payload.get('scope', {}).get('out', []))}</out>
</scope>
<inputs>{json.dumps(payload.get('inputs', {}))}</inputs>
<completion-criteria>
{chr(10).join(f"  {i+1}. {c}" for i, c in enumerate(payload.get('completion_criteria', [])))}
</completion-criteria>
<output-format>
STATUS: DONE | PARTIAL | FAILED
CONFIDENCE: 0-100
SUMMARY: what was accomplished
ARTIFACTS: files created/modified or "none"
ISSUES: anything requiring attention or "none"
</output-format>
<constraints>
  max_turns: {payload.get('constraints', {}).get('max_turns', 20)}
  no_human_confirmation: true
</constraints>
</worker-brief>"""

if __name__ == '__main__':
    DB_PATH.parent.mkdir(parents=True, exist_ok=True)
    main_loop(DB_PATH)
```

---

## 6. Güvenilirlik Katmanları

### 6.1 Doğrulama Hiyerarşisi

```
Level 1 — Schema:     Task payload geçerli JSON mi? Zorunlu alanlar var mı?
Level 2 — Business:   Worker type geçerli mi? Tool listesi doğru mu?
Level 3 — Runtime:    Worker process ayakta mı? result.json yazıldı mı?
Level 4 — Semantic:   STATUS=DONE ama confidence=30 → gerçek başarı mı?
Level 5 — Side-effect: Artifact'lar gerçekten oluşturuldu mu?
```

### 6.2 Hata Hiyerarşisi ve Eylemleri

| Hata Tipi | Tespit | Eylem |
|-----------|--------|-------|
| Worker crash (exit≠0) | exit_code | retry (≤max_retries), sonra failed |
| Silent failure (denials>0) | permission_denials | retry ile daha geniş tool seti |
| Stale worker (timeout) | updated_at | kill + PENDING'e geri al |
| Low confidence (DONE <70) | confidence | retry veya Team Lead'e eskalasyon |
| No result file | dosya yokluğu | failed + retry |
| Budget exceeded | turns_used=maxTurns | PARTIAL olarak kapat, retry gerekebilir |

### 6.3 Circuit Breaker (E-4 adaptasyonu)

```python
# Bir worker type üst üste 5 kez başarısız olursa
if failure_count[worker_type] >= 5:
    # O worker type'ı geçici olarak devre dışı bırak
    # Farklı model veya tool seti dene
    # 60 saniye sonra yeniden dene
```

---

## 7. Dosya Yapısı

```
~/.claude-workers/
  tasks.db              ← SQLite task queue (WAL mode)
  manager.log           ← Manager aktivite logu

/tmp/
  claude-worker-{uuid}/
    brief.txt           ← Worker'a gönderilen brief
    result.json         ← Worker çıktısı (Manager okur, siler)
    stderr.log          ← Worker hata logu
    .claude/            ← Worker'a özel config (izolasyon)

/project/.claude/agents/
  team-lead.md          ← Team Lead agent tanımı
  worker-analyst.md     ← Analyst worker
  worker-builder.md     ← Builder worker
  worker-reviewer.md    ← Reviewer worker
  worker-executor.md    ← Executor worker
  worker-researcher.md  ← Researcher worker
```

---

## 8. Progressif Ölçekleme

Research 006'dan TP kombinasyon kılavuzu:

```
Seviye 1 — MVP:
  Team Lead + sıralı tek worker
  → Manager Script, SQLite, result.json

Seviye 2 — Paralel Workers:
  Team Lead + Fan-Out (3-10 worker aynı anda)
  → MAX_WORKERS artır, parallel spawn

Seviye 3 — Kalite Katmanı:
  Team Lead + Fan-Out + Maker-Checker
  → Kritik görevler için reviewer worker eklenir

Seviye 4 — Pipeline:
  Team Lead + bağımlı worker zinciri
  → Task payload'a depends_on alanı ekle
```

---

## 9. Tasarım Kararları Özeti

| Karar | Seçim | Kaynak |
|-------|-------|--------|
| Permission mode | `dontAsk` (headless safe) | Research 003 |
| Session izolasyonu | `CLAUDE_CONFIG_DIR` per worker | Research 003 |
| IPC | SQLite + result.json hibrit | Research 004 |
| Queue | SQLite WAL mode | Research 005 |
| Race condition | Atomic BEGIN IMMEDIATE claim | Research 005 |
| Stale recovery | 30s timeout + updated_at check | Research 005 |
| Silent failure | `permission_denials` check | Research 003 |
| Worker output | Structured STATUS/CONFIDENCE/SUMMARY | Research 002 (PAT-04, PAT-05) |
| Team lead model | opus | Research 006 (PAT-09) |
| Worker model | sonnet (haiku for analyst) | Research 006 (TP-01) |
| Koordinasyon pattern | Supervisor/Worker (%70 üretim payı) | Research 006 |
| Başlangıç karmaşıklığı | En basit çalışan şeyle başla | Research 001 (A-2) |