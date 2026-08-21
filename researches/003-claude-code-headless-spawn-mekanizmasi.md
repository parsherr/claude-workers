> **Bağlantılar:** [[researches]] | [[CLAUDE]] | [[researches/001-iyi-agent-yazmanin-kurallari]] | [[researches/002-ornek-agentlar-pattern-analizi]] | [[work-management-system-session-notes]]

# Research 003 — Claude Code Subprocess/Headless Spawn Mekanizması

**Tarih:** 2026-08-21  
**Durum:** Tamamlandı  
**Çıktı:** Bu dosya + [[CLAUDE]] güncelleme

---

## Araştırma Sorusu

Worker agent nasıl spawn edilir? Claude Code'u bağımsız, onay beklemeden çalışan bir subprocess olarak nasıl başlatırız? Headless modda ne mümkün, ne değil?

---

## Kaynaklar

1. `claude --help` — Canlı CLI çıktısı (bu makinede çalıştırıldı)
2. [MindStudio — Claude Code Headless Mode](https://www.mindstudio.ai/blog/claude-code-headless-mode-autonomous-agents)
3. [amux.io — Claude Code Headless Self-Hosting Guide](https://amux.io/guides/claude-code-headless/) (403, Rusk ile çekilemedi)
4. [hidekazu-konishi — Claude Code CI/CD Headless Automation](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)
5. [Anthropic Docs — Hosting the Agent SDK](https://code.claude.com/docs/en/agent-sdk/hosting)
6. [morphllm — dangerously-skip-permissions 2026](https://www.morphllm.com/claude-code-dangerously-skip-permissions)
7. [wmedia.es — silent failure trap](https://wmedia.es/en/tips/claude-code-dangerously-skip-permissions-silent-failure)
8. [GitHub Issue #52506 — --permission-mode dontAsk vs --dangerously-skip-permissions](https://github.com/anthropics/claude-code/issues/52506)
9. [augmentcode — Claude Agent SDK](https://www.augmentcode.com/guides/claude-agent-sdk-agent-loops-tool-calls)
10. [buildwithaws — Agent SDK stdio architecture](https://buildwithaws.substack.com/p/inside-the-claude-agent-sdk-from)

---

## Ana Bulgular

### 1. İki Spawn Yöntemi

Worker başlatmak için iki temel yol var:

**Yol A: Claude CLI (`claude -p`)**  
Bash subprocess olarak `claude` binary'sini çalıştırırsın. Basit, direkt, ek dependency yok.

**Yol B: Claude Agent SDK (Python/TypeScript)**  
SDK, arka planda aynı `claude` binary'sini subprocess olarak spawn eder — fakat stdio üzerinden structured API sunar. Daha kontrollü, production için daha uygun.

Her iki yolda da temel mekanizma aynı: `claude` CLI process'i spawn edilir, stdio üzerinden iletişim kurulur.

---

### 2. Temel Headless Komut Yapısı

```bash
# Minimal headless çalıştırma
claude -p "Görevi buraya yaz"

# Production headless (tam kontrol)
claude \
  --print \
  --permission-mode dontAsk \
  --allowedTools "Bash,Read,Write,Edit" \
  --output-format stream-json \
  --max-turns 20 \
  --model sonnet \
  "Görevi buraya yaz"
```

**Kritik flag'ler:**

| Flag | Değer | Açıklama |
|------|-------|----------|
| `-p` / `--print` | — | Headless mod aktif. Prompt al, çalış, çık. |
| `--permission-mode` | `dontAsk` | İzinsiz tool call → auto-deny, asılmaz |
| `--permission-mode` | `bypassPermissions` | Tüm izinleri atla (sandbox ONLY) |
| `--permission-mode` | `auto` | Classifier-based, %93 auto-approve |
| `--allowedTools` | `"Read,Edit,Bash(git *)"` | Sadece belirtilen tool'lara izin |
| `--disallowedTools` | `"Bash(rm *)"` | Belirtilen pattern'ı blokla |
| `--output-format` | `json` / `stream-json` / `text` | Çıktı formatı |
| `--max-turns` | `20` | Hard turn limiti — aşılırsa non-zero exit |
| `--max-budget-usd` | `2.00` | Maliyet tavanı |
| `--model` | `sonnet` / `opus` / `haiku` | Model seçimi |
| `--session-id` | `<UUID>` | Deterministik session ID (idempotency) |
| `--resume` | `<session-id>` | Önceki session'ı devam ettir |
| `--bare` | — | Minimal mod: hook, MCP, CLAUDE.md yok. CI için hız. |
| `--system-prompt` | `"..."` | Inline system prompt override |
| `--no-session-persistence` | — | Session disk'e yazılmaz (sadece -p ile) |
| `--bg` | — | Background agent olarak başlat, hemen dön |

---

### 3. Permission Mode Seçimi — Kritik Karar

Bu konuda önemli bir tuzak var:

**`--dangerously-skip-permissions` headless için YETERSİZ:**
- İlk çalıştırmada interactive confirmation dialog açar
- Headless/TTY-less ortamda asılır (blocking)
- 20 paralel agent başlatıldığında hepsi dialog'da bekler → sıfır iş üretilir

**Doğru headless flag: `--permission-mode dontAsk`**
- İzinsiz tool call → anında auto-deny, asılmaz
- Script non-zero exit olmadan devam eder
- Log'da `permission_denials` array görünür

**Worker spawn için önerilen kombinasyon:**
```bash
claude -p "görev" \
  --permission-mode dontAsk \
  --allowedTools "Bash,Read,Write,Edit,Glob,Grep" \
  --output-format stream-json
```

**Tam güvenli sandbox için (izole container'da):**
```bash
claude -p "görev" \
  --permission-mode bypassPermissions \
  --output-format stream-json
```

---

### 4. Silent Failure Tuzağı

**En tehlikeli anti-pattern:** `claude -p` permission denied olsa bile exit code 0 döner ve `"is_error": false` yazar.

```json
{
  "result": "",
  "is_error": false,
  "permission_denials": ["Bash: denied"],
  "total_cost_usd": 0.001
}
```

Worker "başarılı" göründü, hiçbir şey yapmadı.

**Çözüm:** Her worker çıktısında `permission_denials` array'ini kontrol et. Eğer boş değilse → gerçek başarısızlık.

```bash
output=$(claude -p "görev" --output-format json)
denials=$(echo "$output" | jq '.permission_denials | length')
if [ "$denials" -gt 0 ]; then
  echo "FAILED: permission denied"
  exit 1
fi
```

---

### 5. Output Formatları ve Parsing

**`--output-format json` (tek sonuç):**
```json
{
  "result": "İş tamamlandı",
  "session_id": "uuid-burada",
  "total_cost_usd": 0.05,
  "permission_denials": []
}
```

**`--output-format stream-json` (satır satır JSONL):**
```
{"type":"system","subtype":"init","session_id":"...","model":"..."}
{"type":"assistant","message":{"content":[{"type":"text","text":"..."}]}}
{"type":"result","result":"...","session_id":"..."}
```

**Önerilen:** Stream-json + tee (ham log sakla, parse et):
```bash
claude -p "görev" --output-format stream-json \
  | tee /tmp/worker-$(date +%s).jsonl \
  | jq -r 'select(.type == "result") | .result'
```

---

### 6. Session Yönetimi

**Session ID ile idempotency:**
```bash
# Aynı UUID = aynı logical job (retry güvenli)
JOB_ID="550e8400-e29b-41d4-a716-446655440000"
claude -p "görevi yap" --session-id "$JOB_ID" --output-format json
```

**İki aşamalı pipeline:**
```bash
# Aşama 1
session=$(claude -p "Modülü analiz et" --output-format json | jq -r '.session_id')

# Aşama 2 — aynı context üzerinde devam
claude -p "Bulduklarına göre test yaz" --resume "$session"
```

**State nerede yaşıyor:**

| Durum | Default konum |
|-------|--------------|
| Session transcript | `~/.claude/projects/<encoded-cwd>/` |
| CLAUDE.md memory | `~/.claude/CLAUDE.md` |
| Working dir artifacts | Session'ın cwd'si |

**Kritik:** Container restart'ta bunlar kaybolur. Birden fazla worker aynı `~/.claude/` paylaşırsa session state bozulur.

---

### 7. Paralel Worker Spawn — Isolation Kuralları

**N worker = N subprocess, her biri kendi process tree'sinde.**

```bash
# Paralel 3 worker — bash örneği
for task in task1 task2 task3; do
  claude -p "$(cat tasks/$task.txt)" \
    --permission-mode dontAsk \
    --output-format json \
    > results/$task.json &
done
wait  # hepsini bekle
```

**İzolasyon için zorunlular:**
1. Her worker ayrı `cwd` almalı — `--add-dir` veya process'i o dizinde başlat
2. Her worker ayrı `~/.claude/` (CLAUDE_CONFIG_DIR) kullanmalı — aksi halde session çakışır
3. `--no-session-persistence` kullan — worker'lar session'larını disk'e yazmasın (gerekmedikçe)

```bash
# İzole worker örneği
WORKER_HOME=/tmp/worker-$WORKER_ID
mkdir -p $WORKER_HOME
CLAUDE_CONFIG_DIR=$WORKER_HOME claude -p "görev" \
  --permission-mode dontAsk \
  --output-format json \
  --no-session-persistence
```

---

### 8. `--bare` Modu — Hızlı CI Worker

Hooks, MCP, CLAUDE.md traversal, OAuth/keychain okumalarını atlar:
- Daha hızlı başlangıç
- Deterministik davranış
- CI pipeline için önerilen

```bash
ANTHROPIC_API_KEY=xxx claude -p "görev" --bare \
  --permission-mode dontAsk \
  --output-format json
```

**Trade-off:** `--bare` modda CLAUDE.md okunmaz → worker projeyi bilmez. System prompt ile bağlam sağlanmalı:
```bash
claude -p "görev" --bare \
  --system-prompt "$(cat CLAUDE.md)" \
  --permission-mode dontAsk
```

---

### 9. `--bg` Flag — Background Agent

```bash
claude --bg "Arka planda şu işi yap"
# Hemen döner, agent arka planda çalışır

# Yönetim
claude agents list
claude agents logs <agent-id>
claude agents kill <agent-id>
```

Bu flag worker-pool pattern'ı için potansiyel — manager spawn eder, kontrol eder.

---

### 10. Turn ve Maliyet Kontrolü

```bash
timeout 600 claude -p "görev" \
  --max-turns 20 \
  --max-budget-usd 1.00 \
  --model sonnet \
  --output-format json
```

- `--max-turns` aşılırsa non-zero exit → manager bu sinyali yakalar
- `--max-budget-usd` aşılırsa çalışma durur
- `timeout` ile wall-clock limit → asılı worker'ı öldürür
- `--fallback-model` → primary model unavailable'sa devam eder

---

### 11. Agent SDK — Subprocess Wrapper

Python SDK örneği:
```python
from claude_code_sdk import query, ClaudeAgentOptions

result = await query(
    prompt="Görevi yap",
    options=ClaudeAgentOptions(
        cwd="/work/session-a",
        permission_mode="dontAsk",
        allowed_tools=["Bash", "Read", "Write"],
        max_turns=20,
    )
)
```

SDK'nın yaptığı: aynı `claude` binary'sini subprocess olarak spawn et, stdio üzerinden JSONL mesajlaşma. Üst seviye abstraction sağlar.

---

## Proje İçin Çıkarımlar

### Worker Spawn Stratejisi (Önerilen)

```bash
# Her worker için izole ortam
WORKER_ID=$(uuidgen)
WORKER_HOME=/tmp/claude-worker-$WORKER_ID
mkdir -p $WORKER_HOME/{work,config}

CLAUDE_CONFIG_DIR=$WORKER_HOME/config \
  timeout 600 \
  claude -p "$TASK_CONTENT" \
    --permission-mode dontAsk \
    --allowedTools "Bash,Read,Write,Edit,Glob,Grep,WebSearch" \
    --output-format json \
    --max-turns 30 \
    --model sonnet \
    --no-session-persistence \
  > $WORKER_HOME/result.json
EXIT_CODE=$?

# Silent failure check
DENIALS=$(jq '.permission_denials | length' $WORKER_HOME/result.json)
if [ "$DENIALS" -gt 0 ] || [ "$EXIT_CODE" -ne 0 ]; then
  echo "WORKER $WORKER_ID FAILED"
fi
```

### Kritik Kararlar

| Karar | Seçim | Neden |
|-------|-------|-------|
| Permission mode | `dontAsk` | `--dangerously-skip-permissions` headless'ta asılır |
| Output format | `stream-json` + tee | Hem realtime hem log sakla |
| Session isolation | `CLAUDE_CONFIG_DIR` per worker | `~/.claude/` paylaşımı session bozar |
| Turn limit | `--max-turns 25-30` | Budget guardrail |
| Wall-clock | `timeout 600` | Asılı worker'ı öldür |
| Silent failure | `permission_denials` check | Exit code 0 ≠ başarı |
| Context sağlama | `--system-prompt` ile explicit | `--bare` modda CLAUDE.md okunmaz |

### Açık Sorular (Sonraki Research'e)

- Worker-Manager arası iletişim nasıl olacak? Result JSON'u kim okuyor, nasıl queue'ya geri yazılıyor? → **004'e geçer**
- `--bg` flag'i ile spawn edilen agent'ı nasıl izleriz? `claude agents` komutları yeterli mi?
- `check-completion.py` hook'u `--bare` modda çalışır mı? (Hook discovery atlanıyor)
- Container izolasyonu zorunlu mu? Docker olmadan güvenli paralel çalışma mümkün mü?