> **Bağlantılar:** [[researches]] | [[CLAUDE]] | [[researches/003-claude-code-headless-spawn-mekanizmasi]] | [[researches/005-task-queue-persistence]] | [[work-management-system-session-notes]]

# Research 004 — Worker-Manager İletişim Katmanı (IPC Seçenekleri)

**Tarih:** 2026-08-21  
**Durum:** Tamamlandı  
**Çıktı:** Bu dosya — IPC seçenekleri analizi

---

## Araştırma Sorusu

Worker agent çalışırken Manager ile nasıl iletişim kurar? Sonucu nasıl bildirir? Görev ataması nasıl yapılır? Hangi IPC mekanizması bu proje için en uygun?

---

## Kaynaklar

1. [Anthropic Docs — Agent Teams (v2.1.178)](https://code.claude.com/docs/en/agent-teams)
2. [alexlavaee.me — Parallel Agent Sessions: Infrastructure Gap](https://alexlavaee.me/blog/parallel-agent-sessions-infrastructure-gap/)
3. [Shipyard — Claude Code Multi-Agent Orchestration](https://shipyard.build/blog/claude-code-multi-agent/)
4. [DEV Community — Building Durable Message Queue on SQLite for AI Agent Orchestration](https://dev.to/minnzen/building-a-durable-message-queue-on-sqlite-for-ai-agent-orchestration-335m)
5. [Baeldung — IPC Performance Comparison](https://www.baeldung.com/linux/ipc-performance-comparison)
6. [Apriorit — IPC in Python](https://www.apriorit.com/dev-blog/web-python-ipc-methods)
7. [hidekazu-konishi — Claude Code CI/CD Headless Automation](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)

---

## IPC Seçenekleri Karşılaştırması

### A: Stdio/Pipe (Parent↔Child)

**Nasıl çalışır:** Manager, worker'ı `subprocess.Popen(stdin=PIPE, stdout=PIPE)` ile başlatır. İki yönlü iletişim stdin/stdout üzerinden JSON-NEWLINE akışıyla.

**Artılar:**
- En düşük latency — kernel buffer, disk yok
- Claude Agent SDK tam olarak bu modeli kullanır
- Process exit otomatik temizlenir

**Eksiler:**
- Manager çökürse worker'ın çıktısı kaybolur
- Paralel worker'lar için ayrı pipe çifti gerekir
- Worker manager'dan bağımsız olamaz (parent-child bağı)

**Uygun senaryo:** SDK üzerinden tek worker, kısa görev, manager'ın her zaman ayakta olduğu durum.

---

### B: Dosya Tabanlı (File-Based JSON)

**Nasıl çalışır:** Paylaşılan dizinde JSON dosyaları. Worker görevi bir dosyadan okur, sonucu başka bir dosyaya yazar. Anthropic'in resmi Agent Teams sistemi bu modeli kullanır.

**Anthropic'in seçimi:** `~/.claude/teams/{team}/inboxes/{agent}.json` — interval polling.

**Artılar:**
- En basit implementasyon — bash araçları yeterli
- Manager çöküp kalksa bile dosyalar duruyor (durability)
- Worker'lar tamamen bağımsız process olabilir
- Debug kolay: dosyayı açıp okursun
- Tüm spawn backend'leriyle çalışır (in-process, tmux, iTerm2)

**Eksiler:**
- Her mesajda tüm dosyayı deserialize/serialize: O(N) per message
- Polling interval = latency (100ms-1s)
- Race condition riski → flock + atomic rename gerekir
- Dosya büyüdükçe yavaşlar

**Anthropic neden seçti:** Farklı spawn backend'lerinde (tmux, iTerm2, in-process) ortak yüzey olarak disk her zaman mevcut. Socket veya pipe parent-child ilişkisi gerektirir; farklı terminal pane'lerindeki process'ler için bu imkansız.

---

### C: Unix Domain Socket

**Nasıl çalışır:** Manager bir Unix socket açar (`/tmp/manager.sock`), worker'lar bağlanır. Bidirectional JSON-NEWLINE.

**Artılar:**
- Düşük latency (disk I/O yok)
- Bidirectional: worker → manager push, manager → worker push
- Named pipe'tan daha esnek

**Eksiler:**
- Manager her zaman ayakta olmalı (socket sunucu)
- Worker başlarken manager'ın socket'i açmış olması gerekir (startup sırası)
- tmux pane'lerinde çalışmayabilir (farklı user namespace)

**Uygun senaryo:** Manager'ın long-running daemon olduğu mimari.

---

### D: Named Pipe (FIFO)

**Nasıl çalışır:** `mkfifo /tmp/manager-in /tmp/manager-out`. Worker birer yönlü kanal üzerinden yazar/okur.

**Artılar:**
- Kernel tarafından tamponlanır
- Basit, POSIX garantili

**Eksiler:**
- Unidirectional — iki FIFO gerekir
- Sadece single consumer/producer için güvenli
- Paralel worker'lar için N×2 FIFO → karmaşık

---

### E: SQLite (Queue Database)

**Nasıl çalışır:** Ortak SQLite DB, görevler tabloda. Worker `status='pending'` satırı alır, `status='running'` yapar, biter, `status='done'` yapar. Bu aynı zamanda **005'in konusu** — task queue persistence.

**Artılar:**
- Durability: manager/worker crash = veri kaybolmaz
- Atomic claim: `BEGIN` + `SELECT + UPDATE` race condition'ı önler
- Historical audit trail
- Retry, idempotency, stale worker recovery hepsi native

**Eksiler:**
- External dependency (SQLite library gerekir)
- Polling gerekir (ya da WAL mode + inotify)
- Write-heavy yük için WAL mode açılmalı

**Bu modelde IPC yoktur aslında:** Worker ve Manager aynı DB'yi okur/yazar. İletişim = DB state değişikliği.

---

## Anthropic'in Gerçek Dünya Tercihi — Analiz

Claude Code Agent Teams'in file-based inbox seçimi kasıtlı:

```
Neden dosya?
├── Spawn backend agnostic (tmux, iTerm2, in-process hepsi çalışır)
├── Crash-resilient (manager restart'ta mesajlar kaybolmaz)
├── Debug edilebilir (dosyayı okursun)
└── Zero dependency (extra process/server yok)

Bedeli:
├── O(N) per message (tüm dosya re-serialize)
├── Polling latency (100ms-1s)
└── Race condition → flock zorunlu
```

---

## Tasarım Kararı — Bu Proje İçin

**Önerilen hibrit yaklaşım:**

```
Manager
  │
  ├── Task Queue: SQLite (005'te detay)
  │     task_id, status, payload, worker_id, result
  │
  ├── Worker Spawn: claude -p subprocess (003'te detay)
  │     Her worker: CLAUDE_CONFIG_DIR izole, --output-format json
  │
  └── Sonuç Toplama: İKİ seçenek

SEÇENEK 1 — Stdout Capture (Basit)
  Worker stdout'u --output-format json ile yakala
  Manager, subprocess.communicate() veya tee ile okur
  Sonucu SQLite'a yaz
  
  + En basit implementasyon
  - Manager worker bitene kadar bloklar (veya thread gerekir)

SEÇENEK 2 — File Result (Robust)
  Her worker ayrı result file yazar: /tmp/worker-{id}/result.json
  Manager bu dosyayı polling veya inotify ile izler
  Worker biter, Manager dosyayı okur, SQLite günceller
  
  + Manager bloklamaz
  + Worker crash → dosya eksik = açık sinyal
  + Debug: dosyayı elle okuyabilirsin
```

**Önerilen: Seçenek 2 + SQLite queue kombinasyonu**

```
Görev akışı:
1. Manager → SQLite'a INSERT task (status='pending')
2. Manager → Worker spawn: claude -p <task> > /tmp/worker-{id}/result.json
3. Worker çalışır, tamamlar, result.json yazar, exit 0
4. Manager result.json okur, SQLite'ı günceller (status='done', result=...)
5. Worker dizini temizlenir
```

---

## Mesaj Formatları

### Görev payload (Manager → Worker)
```json
{
  "task_id": "uuid",
  "type": "code_review",
  "prompt": "Şu dosyayı incele: ...",
  "context": {
    "cwd": "/project",
    "files": ["src/auth.py"]
  },
  "constraints": {
    "max_turns": 20,
    "allowed_tools": ["Read", "Grep", "Glob"]
  }
}
```

### Sonuç payload (Worker → Manager)
```json
{
  "task_id": "uuid",
  "status": "done",
  "result": "...",
  "permission_denials": [],
  "total_cost_usd": 0.05,
  "session_id": "claude-session-uuid",
  "exit_code": 0,
  "completed_at": "2026-08-21T12:00:00Z"
}
```

### Hata payload
```json
{
  "task_id": "uuid",
  "status": "failed",
  "error": "max_turns_exceeded",
  "partial_result": "...",
  "permission_denials": ["Bash: denied"],
  "exit_code": 1,
  "completed_at": "2026-08-21T12:00:00Z"
}
```

---

## Veri Akışı — Sequence

```
Manager                           Worker (claude -p)
  │                                    │
  │──[SQLite INSERT task pending]──►   │
  │                                    │
  │──[spawn subprocess]────────────────►
  │                               [reads task payload]
  │                               [executes claude -p]
  │                               [writes result.json]
  │                               [exit 0/1]
  │◄──────────────────────────────[process exit event]
  │                                    │
  │──[reads result.json]               │
  │──[SQLite UPDATE status=done]       │
  │──[cleanup /tmp/worker-{id}/]       │
```

---

## Cross-Session Messaging (Alternatif)

Anthropic'in `code.claude.com/docs/en/cross-session-messaging` sayfası: iki bağımsız Claude session'ı arasında message gönderimi. Agent Teams'den farklı — `Teammate` tool gerektirmez, herhangi iki session arasında çalışır.

Bu projede şimdilik gerekli değil — Manager bir Claude session değil, bizim yazdığımız bir orkestratör script olacak.

---

## Önemli Uyarılar

1. **`~/.claude/` paylaşımı** — Birden fazla worker aynı config dizinini paylaşırsa session çakışır. Her worker: `CLAUDE_CONFIG_DIR=/tmp/worker-{id}/.claude`
2. **Stale worker detection** — Worker'ın process'i var ama result.json yok + timeout geçti → öldür, task'ı `failed` yap, retry queue'ya gönder
3. **Silent failure** — `exit_code=0` ama `permission_denials` dolu → gerçek başarısızlık (003'ten)
4. **Paralel write race** — Birden fazla worker aynı SQLite DB'ye yazıyorsa WAL mode + transaction zorunlu