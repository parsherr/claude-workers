> **Bağlantılar:** [[researches]] | [[CLAUDE]] | [[agent-generator/core-idea]] | [[agent-generator/rules]]

# Research 001 — İyi Agent Yazmanın Kuralları

**Tarih:** 2026-08-21  
**Durum:** Tamamlandı  
**Çıktı:** `agent-generator/rules.md`

---

## Araştırma Sorusu

İyi bir Claude/LLM agent yazmak ne demektir? Hangi kurallar, ilkeler ve kararlar bir agent'ı güvenilir, tamamlayıcı ve bakımı kolay kılar?

Odak alanları:
- **System prompt:** Nasıl yapılandırılmalı? Ne içermeli?
- **Tool seçimi:** Hangi araçlar, nasıl tanımlanmalı?
- **Completion:** Agent işi "bitirdi" dediğinde gerçekten bitmiş mi?
- **Hata yönetimi:** Compounding errors, silent failures, recovery

---

## Kaynaklar

1. [Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) — Resmi Anthropic rehberi, 7 temel pattern
2. [arXiv 2604.14228 — Dive into Claude Code](https://arxiv.org/abs/2604.14228) — Claude Code'un tersine mühendislik analizi, 13 tasarım ilkesi
3. [Confident AI — LLM Agent Evaluation 2026](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide) — Completion detection metrikleri
4. [ValueStream AI — Error Handling Patterns 2026](https://valuestreamai.com/blog/ai-error-handling-patterns-2026) — 8 hata yönetimi pattern'ı
5. [Runyard — AI Agent System Prompts Guide](https://runyard.io/blog/ai-agent-system-prompts-guide) — System prompt best practices
6. [Paxrel — AI Agent Prompt Engineering 2026](https://paxrel.com/blog-ai-agent-prompts) — 10 çalışan pattern
7. [Inflectra — Prompt Engineering for AI Agents](https://www.inflectra.com/Ideas/Topic/AI-Agent-Prompt-Engineering.aspx) — Completion criteria odaklı
8. [Virtido — Agentic Workflow Patterns](https://virtido.com/blog/agentic-workflows-patterns-best-practices-enterprise) — Enterprise patterns
9. [Teamday — Agentic Coding Guide 2026](https://www.teamday.ai/blog/complete-guide-agentic-coding-2026) — Claude Code odaklı

---

## Ana Bulgular

### 1. System Prompt = Operasyonel Politika Belgesi

System prompt bir "kişilik taslağı" değil, **çalışma talimatnamesidir (runbook)**. Şunları içermeli:

- **Rol tanımı:** Agent ne yapıyor, kimin adına, hangi kapsamda?
- **Başarı kriterleri:** "Bitti" nasıl görünür? Ölçülebilir mi?
- **Tool kullanım hiyerarşisi:** Hangi durumlarda hangi tool kullanılmalı, öncelik sırası nedir?
- **Eskalasyon koşulları:** Ne zaman durup insan onayı istenmeli?
- **Sınırlar ve yasaklar:** Agent neyi yapmamalı?
- **Output formatı:** Çıktı nasıl görünmeli, nereye yazılmalı?

Claude için özel not: XML tag'leri ve açık düşünme blokları en iyi sonucu verir.

**Kritik kural:** System prompt, bir chatbot için değil bir **otonom süreç** için yazılır. "Ne söyle" değil "ne zaman ne yap" sorusuna cevap verir.

---

### 2. Araçlar = Agent-Computer Interface (ACI)

Araç tasarımı, insan-bilgisayar arayüzü (HCI) kadar önem taşır. Anthropic'in SWE-bench deneyimi: **tool optimizasyonu, prompt optimizasyonundan daha fazla zaman aldı.**

Temel kurallar:
- **Dokümantasyon:** Her tool, bir junior developer için yazılmış docstring gibi belgelenmeli. Örnek kullanım, edge case'ler, format gereksinimleri.
- **Poka-yoke:** Argümanları hata yapmayı zorlaştıracak şekilde tasarla. (Örnek: relative path → absolute path değişikliği hata sınıfını tamamen ortadan kaldırdı.)
- **Az ama keskin:** Geniş toolset < az sayıda net sınırlı tool. Overlapping sorumluluklar agent'ı kafa karıştırır.
- **Explicit hiyerarşi:** "Önce cache'e bak, sonra web'e git" gibi açık öncelik kuralları prompt'ta yazılmalı.
- **Pre-filtering:** İzin verilmeyen tool'lar model'in görmediği araçlara dönüştürülmeli — bu Claude Code'un "deny-first" ilkesidir.

---

### 3. Completion Detection — "Bitti mi?" Sorusu

En tehlikeli agent başarısızlıkları **başarı gibi görünür**: 200 OK döner, ama iş yanlış yapılmıştır.

**Claude Code'un 5 stop koşulu:**
1. Tool kullanımı yok (metin-only çıktı) → birincil stop sinyali
2. `maxTurns` limitine ulaşıldı
3. Context overflow (tüm recovery denemelerinden sonra)
4. Hook müdahalesi (`PostToolUse` hook `hook_stopped_continuation` set eder)
5. Explicit abort sinyali

**Completion değerlendirme katmanları:**
- **Deterministik:** Tool adı doğru mu? Argümanlar geçerli mi? Beklenen output formatında mı?
- **LLM-judge:** Task gerçekten tamamlandı mı? (Trajectory üzerinden LLM değerlendirmesi)
- **Trace inspection:** Loop derinliği spike'ı var mı? Beklenmedik tool çağrı sayısı?
- **Periyodik human review:** Metriklerle yakalanmayan drift için

**Bizim hook yaklaşımımız (`check-completion.py`):** LLM semantic check ile "Gerçekten bitti mi?" sorusu sorulur. "Hayır" → `exit 2` → Claude devam ettirilir. Bu, Anthropic'in PostToolUse hook müdahalesi ilkesiyle örtüşüyor.

---

### 4. Hata Yönetimi — 8 Kritik Pattern

#### 4.1 Retry + Exponential Backoff + Jitter
```
wait = min(base_delay * 2^attempt + jitter, max_delay)
```
- Base: 1-2s, max: 60-120s, max attempt: 5-7
- 429/500-504/timeout: retry. 400/401/403: fail fast.
- `Retry-After` header'ına uyu.

#### 4.2 Circuit Breaker (Kalite Genişlemeli)
Sadece HTTP error'da değil, **semantic sinyal**de de tetiklenir:
- Schema validation fail > %15 → breaker aç
- 5 ardışık hata → breaker aç
- Düşük kaliteli çıktı oranı yüksekse → fallback'e geç

#### 4.3 Fallback Chain
Provider'ı değiştir (sadece model değil) — aynı infrastructure paylaşılıyor olabilir.

#### 4.4 Validation Gates (3 Katman) — En Kritik
**Tool çağrısından ÖNCE** çalıştırılmalı, sonra değil:
1. Schema (Pydantic/JSON Schema): format doğru mu?
2. Business logic: entity var mı, değer mantıklı mı?
3. Safety/Policy: PII, compliance, prompt injection?

#### 4.5 Idempotent Saga Pattern
Multi-step workflow'larda naive retry, tamamlanan adımları tekrar çalıştırır (çift email, çift ödeme). Çözüm:
- Her adımdan ÖNCE intent'i durable store'a yaz
- Idempotency key: `workflow_id + step_number`
- Her adımın bir compensation (geri alma) aksiyonu var

#### 4.6 Budget Guardrails
4 boyut: token, cycle (LLM çağrı sayısı), wall-clock time, maliyet (USD).
- Tipik agent: 5-15 adım. Hard ceiling: 25-30.
- Budget aşılınca: kısmen tamamlanmış sonucu döndür, logla, alert et.

#### 4.7 Graceful Degradation (5 Seviye)
1. Cache response
2. Reduced-capability response (daha basit model)
3. Partial result (tamamlananlar + eksikler)
4. Manual fallback (insan kuyruğu)
5. Informed failure (ne bozuldu, kullanıcı ne yapabilir)

#### 4.8 Human Escalation Design
Eskalasyon bir **tasarım kararı**, başarısızlık değil. Tetikleyiciler:
- 3 retry sonrası aynı validation hatası
- Safety/policy ihlali
- Yüksek değerli geri alınamaz aksiyon
- Kullanıcı açıkça istedi

---

### 5. Mimari İlkeler (Claude Code'dan)

**"98.4% Harness" bulgusu:** Claude Code'un sadece %1.6'sı AI karar mantığı. %98.4'ü güvenilir çalışmayı sağlayan operasyonel harness. **Temel model yetenekleri yakınsadıkça, güvenilirliği belirleyen deterministik mühendislik altyapısıdır.**

**13 Tasarım İlkesinden öne çıkanlar:**
1. **Deny-first:** Tanınmayan aksiyonlar bloklanır veya eskaledir.
2. **Reversibility-weighted risk:** Read-only paralel, state-modifying serialize.
3. **Context as scarce resource:** Context window bir bütçedir, çöp deposu değil.
4. **Isolated subagent boundaries:** Subagent sadece özet döndürür, tam context değil.
5. **Values over rules:** Katı kurallar yerine bağlamsal yargı + deterministik guardrail.
6. **Minimal scaffolding, maximal harness:** Model kararları verir, harness yürütür.

---

### 6. ReAct + Reflection Loop

Modern agent'lar tek pattern'la değil, **katmanlı pattern'larla** çalışır:

```
Plan → ReAct (Reason→Act→Observe) → Reflection (Critique→Improve) → Verify
```

- **ReAct:** Adım adım: düşün → tool çağır → sonucu gözlemle → tekrar düşün
- **Reflection:** Üret → eleştir → iyileştir döngüsü. CRITIC framework: external tool ile doğrulama, %10-30 accuracy artışı.
- **Self-consistency:** Birden fazla yol dene, çoğunluk/en iyi sonucu al.
- **Planning:** Karmaşık task'ta önce plan yap, sonra yürüt.

**Üretim gerçeği:** Hiçbir production sistem tek pattern kullanmıyor. Katmanlı kullanım zorunlu.

---

### 7. Context Yönetimi = Yeni Temel Beceri

2026'da "agent başarısızlıklarının çoğu model hatası değil, **context hatası**dır":
- Yanlış belgeler alındı
- Context window'a çok fazla history doldu
- Tool tanımları unutuldu

LangChain'in 4 stratejisi:
1. **Write:** Context'i dışarıya persist et
2. **Select:** RAG ile ilgili olanı getir
3. **Compress:** Özetle ve kompakt hale getir
4. **Isolate:** Farklı agent'lar için ayrı context

---

### 8. Değerlendirme Metrikleri

Agent'lar 3 seviyede değerlendirilmeli:
1. **End-to-end:** Task başarıyla tamamlandı mı?
2. **Trajectory:** Yol verimli ve mantıklı mıydı?
3. **Component:** Hangi tool/retriever/subagent bozuldu?

**Temel metrikler:** Task completion, tool correctness, argument correctness, step efficiency, plan quality, plan adherence, reasoning relevancy, reasoning coherence.

**Ne zaman deterministik, ne zaman LLM-judge:**
- Deterministik: Tool adı, parametreler, beklenen output değerleri
- LLM-judge: Task completion, reasoning quality, plan adherence

---

## Çıkarımlar Bu Proje İçin

1. `check-completion.py` hook'umuz doğru yönde — Anthropic'in PostToolUse hook completion detection ilkesiyle örtüşüyor.
2. Worker agent'larımızın **idempotent saga pattern** kullanması gerekiyor — aynı görev iki kez çalışırsa güvenli olmalı.
3. **Validation gate** öncelik: tool çağrısından önce göreve uygunluk doğrulanmalı.
4. **Budget guardrails** kritik: worker'ların maximum turn/token limiti olmalı.
5. System prompt'lar "operasyonel politika belgesi" olarak yazılmalı — sadece görev değil, completion kriterleri + eskalasyon koşulları da içermeli.
6. **Context isolation:** Her worker'ın kendi context window'u olmalı (Claude Code subagent izolasyonu bu yüzden var).
7. **Harness > Model:** Worker altyapısına (queue, state machine, monitoring) model tuning'inden daha fazla yatırım yapılmalı.