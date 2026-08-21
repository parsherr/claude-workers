> **Bağlantılar:** [[researches]] | [[CLAUDE]] | [[researches/004-worker-manager-ipc-iletisim]] | [[work-management-system-session-notes]]

# Research 005 — Task Queue Persistence (SQLite vs Dosya Tasarımı)

**Tarih:** 2026-08-21  
**Durum:** Tamamlandı  
**Çıktı:** Bu dosya — task queue şema ve implementasyon kararı

---

## Araştırma Sorusu

Görevler nerede ve nasıl saklanır? Durum makinesi nasıl tasarlanır? Race condition nasıl önlenir? SQLite mi, düz dosya mı?

---

## Kaynaklar

1. [DEV — Building Durable Message Queue on SQLite for AI Agent Orchestration](https://dev.to/minnzen/building-a-durable-message-queue-on-sqlite-for-ai-agent-orchestration-335m)
2. [reflect.run — Implementing a Task Queue in SQL](https://reflect.run/articles/sql-queue/)
3. [DEV — Why I Built a Job Queue With SQLite Instead of Redis](https://dev.to/d_security/why-i-built-a-job-queue-with-sqlite-instead-of-redis-and-what-i-learned-4f05)
4. [GitHub — litements/litequeue](https://github.com/litements/litequeue)
5. [knowledgelib.io — Distributed Job Queue System Design](https://knowledgelib.io/software/system-design/job-task-queue/2026)
6. [michalsniezko — Database-Backed Task Queue](https://michalsniezko.github.io/backend-patterns-optimization/database-backed-task-queue.html)
7. [DEV — Crash-Safe JSON Atomic Writes](https://dev.to/constanta/crash-safe-json-at-scale-atomic-writes-recovery-without-a-db-3aic)
8. [SQLite Renaissance 2026](https://pockit.tools/blog/sqlite-renaissance-turso-d1-libsql-production-guide/)

---

## Seçenek Karşılaştırması

### Seçenek A: Düz JSON Dosyası

```
tasks/
  queue.json         ← pending görevler
  running.json       ← çalışan görevler
  done/
    {task-id}.json   ← tamamlanan görevler
  failed/
    {task-id}.json   ← başarısız görevler
```

**Artılar:**
- Zero dependency
- Debug edilebilir (cat ile okursun)
- Git ile version-controllable

**Eksiler:**
- Race condition: atomic write için flock + mktemp + mv zorunlu
- Query yok: "5 dakikadır çalışan task'lar" için jq pipeline
- Büyüyen queue → tüm dosyayı parse etmek gerekir
- Concurrent worker'lar için file locking karmaşık

**Sonuç:** Küçük ölçek için uygun (1-3 worker, 10'dan az task). Büyük ölçekte kırılgan.

---

### Seçenek B: SQLite

**Artılar:**
- ACID transactionlar — race condition native olarak önlenir
- Queries: `status='pending' ORDER BY created_at LIMIT 1`
- `sqlite_sequence` ile audit trail
- Python'da `import sqlite3` — zero extra dependency
- WAL mode: concurrent reader + writer
- Stale worker recovery: `WHERE status='running' AND updated_at < datetime('now','-10 minutes')`

**Eksiler:**
- Tek writer bottleneck (WAL mode ile hafifler)
- Binary format — doğrudan `cat` ile okuyamazsın (ama `sqlite3 db.sql ".dump"` çalışır)

**Sonuç:** Bu proje için doğru seçim. External dependency yok, transaction garantileri var.

---

## Task State Machine

```
                  ┌──────────┐
                  │  PENDING │  ← Initial state
                  └────┬─────┘
                       │ worker claims
                  ┌────▼─────┐
                  │ RUNNING  │  ← Worker active
                  └────┬─────┘
           ┌───────────┴───────────┐
           │                       │
      ┌────▼─────┐           ┌─────▼────┐
      │   DONE   │           │  FAILED  │
      └──────────┘           └─────┬────┘
                                   │ retry_count < max_retries
                              ┌────▼─────┐
                              │  PENDING │  ← Retry
                              └──────────┘
```

**Durum Geçişleri:**
- `PENDING → RUNNING`: Worker görev alır (atomic claim)
- `RUNNING → DONE`: Worker başarıyla tamamlar
- `RUNNING → FAILED`: Worker hata bildirir veya timeout
- `FAILED → PENDING`: Retry hakkı varsa (retry_count < max_retries)
- `RUNNING → PENDING`: Stale recovery (worker süreciği ölmüş)

---

## SQLite Şeması

```sql
-- WAL mode: concurrent read + single write
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;

-- Ana görev tablosu
CREATE TABLE IF NOT EXISTS tasks (
    id              TEXT PRIMARY KEY,           -- UUID
    type            TEXT NOT NULL,              -- 'code_review', 'file_write', vs.
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK(status IN ('pending','running','done','failed')),
    priority        INTEGER NOT NULL DEFAULT 0, -- Yüksek = önce işle
    payload         TEXT NOT NULL,              -- JSON: görev detayı
    result          TEXT,                       -- JSON: worker sonucu
    error           TEXT,                       -- Hata mesajı (failed durumda)
    worker_id       TEXT,                       -- Hangi worker çalıştırıyor
    retry_count     INTEGER NOT NULL DEFAULT 0,
    max_retries     INTEGER NOT NULL DEFAULT 3,
    claim_id        INTEGER NOT NULL DEFAULT 0, -- Stale claim tespiti
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT NOT NULL DEFAULT (datetime('now')),
    started_at      TEXT,                       -- RUNNING'e geçtiğinde
    completed_at    TEXT,                       -- DONE veya FAILED
    timeout_seconds INTEGER NOT NULL DEFAULT 600 -- Wall-clock limit
);

-- Claim index: pending task'ları hızlı bul
CREATE INDEX IF NOT EXISTS idx_tasks_status_priority
    ON tasks(status, priority DESC, created_at ASC);

-- Worker index: belirli worker'ın task'larını bul
CREATE INDEX IF NOT EXISTS idx_tasks_worker
    ON tasks(worker_id, status);

-- Stale task recovery: timeout geçmiş running task'lar
CREATE INDEX IF NOT EXISTS idx_tasks_stale
    ON tasks(status, updated_at)
    WHERE status = 'running';

-- Audit log (opsiyonel ama önerilen)
CREATE TABLE IF NOT EXISTS task_events (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id     TEXT NOT NULL REFERENCES tasks(id),
    event       TEXT NOT NULL,   -- 'claimed', 'completed', 'failed', 'retry', 'stale'
    worker_id   TEXT,
    detail      TEXT,            -- JSON
    occurred_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

## Kritik Operasyonlar

### 1. Atomic Task Claim (Race Condition Önlemi)

Naive yaklaşım çalışmaz — iki worker aynı task'ı alabilir:
```sql
-- YANLIŞ: İki worker aynı satırı alabilir
SELECT id FROM tasks WHERE status='pending' LIMIT 1;
UPDATE tasks SET status='running' WHERE id=?;
```

Doğru yaklaşım — tek atomik operasyon:
```sql
-- DOĞRU: SQLite transaction içinde atomic claim
BEGIN IMMEDIATE;

SELECT id, payload, claim_id
FROM tasks
WHERE status = 'pending'
ORDER BY priority DESC, created_at ASC
LIMIT 1;

-- Satır varsa:
UPDATE tasks
SET status = 'running',
    worker_id = :worker_id,
    claim_id = claim_id + 1,
    started_at = datetime('now'),
    updated_at = datetime('now')
WHERE id = :task_id
  AND status = 'pending';  -- Double-check: başka worker almamış olsun

COMMIT;
```

Python ile:
```python
import sqlite3
import uuid

def claim_task(db_path: str, worker_id: str) -> dict | None:
    conn = sqlite3.connect(db_path, isolation_level=None)
    conn.row_factory = sqlite3.Row
    try:
        conn.execute("BEGIN IMMEDIATE")
        task = conn.execute(
            """SELECT id, payload, claim_id FROM tasks
               WHERE status = 'pending'
               ORDER BY priority DESC, created_at ASC
               LIMIT 1"""
        ).fetchone()
        if not task:
            conn.execute("ROLLBACK")
            return None
        conn.execute(
            """UPDATE tasks SET
               status='running', worker_id=?, claim_id=claim_id+1,
               started_at=datetime('now'), updated_at=datetime('now')
               WHERE id=? AND status='pending'""",
            (worker_id, task["id"])
        )
        conn.execute("COMMIT")
        return dict(task)
    except Exception:
        conn.execute("ROLLBACK")
        raise
    finally:
        conn.close()
```

### 2. Task Tamamlama

```python
def complete_task(db_path: str, task_id: str, claim_id: int, result: dict):
    conn = sqlite3.connect(db_path)
    conn.execute(
        """UPDATE tasks SET
           status='done', result=?, completed_at=datetime('now'),
           updated_at=datetime('now')
           WHERE id=? AND claim_id=?""",
        (json.dumps(result), task_id, claim_id)
    )
    # claim_id check: stale worker eski claim'i kapatamaz
    conn.commit()
    conn.close()
```

### 3. Hata Bildirimi + Retry Logic

```python
def fail_task(db_path: str, task_id: str, claim_id: int, error: str):
    conn = sqlite3.connect(db_path)
    conn.execute(
        """UPDATE tasks SET
           status = CASE
               WHEN retry_count < max_retries THEN 'pending'
               ELSE 'failed'
           END,
           retry_count = retry_count + 1,
           error = ?,
           worker_id = NULL,
           updated_at = datetime('now')
           WHERE id = ? AND claim_id = ?""",
        (error, task_id, claim_id)
    )
    conn.commit()
    conn.close()
```

### 4. Stale Worker Recovery

```python
def recover_stale_tasks(db_path: str, timeout_multiplier: float = 1.5):
    """Timeout geçmiş RUNNING task'ları PENDING'e döndür."""
    conn = sqlite3.connect(db_path)
    conn.execute(
        """UPDATE tasks SET
           status = 'pending',
           worker_id = NULL,
           updated_at = datetime('now')
           WHERE status = 'running'
           AND datetime(updated_at, '+' || CAST(timeout_seconds * ? AS INT) || ' seconds')
               < datetime('now')""",
        (timeout_multiplier,)
    )
    recovered = conn.total_changes
    conn.commit()
    conn.close()
    return recovered
```

Manager bu fonksiyonu düzenli çalıştırır (örn. her 30 saniyede bir).

---

## Task Ekleme (Producer)

```python
def enqueue_task(db_path: str, task_type: str, payload: dict,
                 priority: int = 0, max_retries: int = 3,
                 timeout_seconds: int = 600) -> str:
    task_id = str(uuid.uuid4())
    conn = sqlite3.connect(db_path)
    conn.execute(
        """INSERT INTO tasks
           (id, type, payload, priority, max_retries, timeout_seconds)
           VALUES (?, ?, ?, ?, ?, ?)""",
        (task_id, task_type, json.dumps(payload),
         priority, max_retries, timeout_seconds)
    )
    conn.commit()
    conn.close()
    return task_id
```

---

## Manager Polling Loop

```python
import time, subprocess, json, os

def manager_loop(db_path: str, worker_script: str, max_workers: int = 3):
    active_workers: dict[str, subprocess.Popen] = {}

    while True:
        # 1. Stale recovery
        recover_stale_tasks(db_path)

        # 2. Tamamlanan worker'ları topla
        for task_id in list(active_workers):
            proc = active_workers[task_id]
            if proc.poll() is not None:
                result_path = f"/tmp/worker-{task_id}/result.json"
                if os.path.exists(result_path):
                    result = json.load(open(result_path))
                    if result.get("permission_denials") or proc.returncode != 0:
                        fail_task(db_path, task_id, ..., "permission_denied_or_crash")
                    else:
                        complete_task(db_path, task_id, ..., result)
                else:
                    fail_task(db_path, task_id, ..., "no_result_file")
                del active_workers[task_id]

        # 3. Yeni task al (kapasite varsa)
        while len(active_workers) < max_workers:
            task = claim_task(db_path, worker_id=f"manager-worker-{os.getpid()}")
            if not task:
                break
            proc = spawn_worker(task)
            active_workers[task["id"]] = proc

        time.sleep(1)  # 1 saniye polling
```

---

## WAL Mode — Concurrent Erişim

```sql
-- Manager başlangıcında bir kez çalıştır
PRAGMA journal_mode=WAL;
PRAGMA synchronous=NORMAL;  -- WAL ile yeterince güvenli, daha hızlı
PRAGMA busy_timeout=5000;   -- Lock bekle, hata verme
```

WAL mode ile:
- Birden fazla reader aynı anda okuyabilir
- Tek writer, reader'ları bloklamaz
- Crash recovery: WAL log yeniden uygulanır, veri kaybolmaz

---

## Bu Proje İçin Sonuç

**Seçim: SQLite (WAL mode) + /tmp/worker-{id}/result.json hibrit**

| Bileşen | Seçim | Neden |
|---------|-------|-------|
| Task queue | SQLite | ACID, query, race condition safe |
| Worker-Manager iletişim | result.json file | Manager bloklamaz, crash-resilient |
| Polling | 1 saniye interval | Yeterli latency, CPU dostu |
| Stale recovery | 30 saniye cron | `updated_at` + `timeout_seconds` |
| Idempotency | `claim_id` check | Stale worker eski claim'i kapatamaz |

**Dosya yapısı:**
```
~/.claude-workers/
  tasks.db           ← SQLite task queue (WAL mode)

/tmp/
  worker-{uuid}/
    result.json      ← Worker çıktısı (Manager okur, siler)
    .claude/         ← Worker'a özel config dizini (izolasyon)
```