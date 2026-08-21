> **Bağlantılar:** [[researches]] | [[CLAUDE]] | [[agent-generator/team-patterns]] | [[researches/003-claude-code-headless-spawn-mekanizmasi]] | [[researches/004-worker-manager-ipc-iletisim]]

# Research 006 — Multi-Agent Koordinasyon Pattern'ları

**Tarih:** 2026-08-21  
**Durum:** Tamamlandı  
**Çıktı:** `agent-generator/team-patterns.md`

---

## Araştırma Sorusu

Birden fazla Claude agent birlikte nasıl çalışır? Hangi koordinasyon pattern'ları üretim ortamında işe yarıyor? Supervisor/Worker, Peer-to-Peer, Pipeline, Fan-Out — ne zaman hangisi?

---

## Kaynaklar

1. [DigitalApplied — Multi-Agent Orchestration: 5 Patterns That Work](https://www.digitalapplied.com/blog/multi-agent-orchestration-5-patterns-that-work)
2. [Beam AI — 6 Multi-Agent Orchestration Patterns for Production](https://beam.ai/agentic-insights/multi-agent-orchestration-patterns-production)
3. [Openlayer — Multi-Agent System Architecture Guide](https://www.openlayer.com/blog/multi-agent-system-architecture-guide)
4. [decodethefuture — Multi-Agent Systems Explained 2026](https://decodethefuture.org/en/multi-agent-systems-explained/)
5. [Anthropic Docs — Agent Teams](https://code.claude.com/docs/en/agent-teams)
6. [AddyOsmani — Claude Code Swarms](https://addyosmani.com/blog/claude-code-agent-teams/)
7. [promptessor — Claude Code Agent Teams 2026](https://promptessor.com/blog/claude-code-agent-teams-examples-and-multi-agent-workflows-for-parallel-development-in-2026)
8. [atlan — Multi-Agent System Orchestration](https://atlan.com/know/multi-agent-system-orchestration/)

---

## Kritik Ekonomi Bulgusu

**Multi-agent her zaman daha iyi değildir.** Princeton NLP 2026 bulgusu:
- Single agent, eşdeğer araç ve context verildiğinde benchmarked task'ların **%64'ünde** multi-agent'ı geçiyor veya eşit performans gösteriyor.
- Multi-agent yaklaşık **+2 puan accuracy** sağlar, ama **yaklaşık 2 katı maliyet**.

**Token overhead gerçeği:**
- Bağımsız peer-to-peer multi-agent: **+%58 token overhead**
- Supervisor/orchestrator merkezi: **+%285 token overhead**
- Multi-agent debate: **+%150** (2.5× maliyet)

**Sonuç:** Multi-agent yalnızca görev gerçekten şunlardan yararlandığında kullanılır:
1. Paralelizm (bağımsız alt görevler)
2. Uzmanlaşma (farklı domain expertise)
3. Eleştiri/doğrulama (birbirinin hatalarını yakalayan agent'lar)

---

## 6 Koordinasyon Pattern'ı — Detaylı Analiz

### Pattern 1: Supervisor / Orchestrator-Worker

**Mimarisi:**
```
        ┌─────────────────┐
        │   Supervisor    │  ← Planlama, dağıtım, sentez
        │   (opus model)  │
        └────────┬────────┘
    ┌───────────┬┴───────────┐
    ▼           ▼            ▼
┌───────┐  ┌───────┐  ┌───────┐
│ W-1   │  │ W-2   │  │ W-3   │  ← Uzman worker'lar
│(code) │  │(test) │  │(docs) │    (sonnet/haiku)
└───────┘  └───────┘  └───────┘
```

**Ne zaman kullanılır:**
- Görev farklı uzmanlık alanları gerektiriyor
- Alt görev yapısı önceden bilinebilir
- Tek sorumluluk merkezi gerekli

**2026 üretim payı:** ~%70 — en yaygın pattern

**Avantajları:**
- %40-60 maliyet tasarrufu (ucuz worker modeller)
- Tek orchestrator = açık hesap verebilirlik
- Native framework desteği en geniş

**Dezavantajları:**
- Supervisor single point of failure
- 4+ worker'da context overflow riski
- Over-delegation: supervisor alt görevleri fazla böler → worker'lar eksik döndürür → sonsuz yeniden dağıtım

**Bu proje için:** Ana pattern. Manager = supervisor (bizim script), Worker = claude subprocess.

---

### Pattern 2: Sequential Pipeline

**Mimarisi:**
```
[Agent A] → çıktı → [Agent B] → çıktı → [Agent C] → çıktı → [Sonuç]
```

**Ne zaman kullanılır:**
- Katı sıralı bağımlılıklar (her adım öncekinin tam çıktısını gerektirir)
- ETL, belge işleme, hukuki iş akışları
- Debug edilebilirlik öncelik

**Gerçek örnek:** 4 agent'lı hukuki sözleşme: template seçimi → madde özelleştirme → uyumluluk → risk değerlendirmesi

**Avantajları:** En basit, en debuggable. Durum tek yönde akar.

**Dezavantajları:**
- Latency = tüm aşamaların toplamı
- Cascade failure: ilk aşama hatası hepsini kirletir
- 4 agent pipeline: ~950ms koordinasyon overhead, ~500ms gerçek işlem

**Anti-pattern:** Paralel çalışabilecek aşamaları pipeline'a sokmak.

---

### Pattern 3: Fan-Out / Fan-In (Paralel)

**Mimarisi:**
```
                ┌─────────────┐
                │ Dispatcher  │
                └──────┬──────┘
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   [Agent A]       [Agent B]       [Agent C]  ← Paralel
        └───────────────┬───────────────┘
                        ▼
                  [Aggregator]
```

**Ne zaman kullanılır:**
- 4+ bağımsız alt görev
- Multi-perspektif analiz (farklı kaynaklardan araştırma)
- Paralel kod review (güvenlik, performans, stil ayrı agent'lar)

**Avantajları:**
- Duvar saati süresi = en yavaş branch (toplamı değil)
- %75 latency azalması mümkün

**Dezavantajları:**
- Rate limit kolektif sorun: 15 paralel agent bireysel limitler içinde olsa da paylaşılan kapasiteyi aşabilir
- Partial failure kararı: bir branch hata verince → hepsini durdur mu? Kısmi sonuç mu döndür? Retry mi?
- Shared state race condition: N agent × (N-1) çakışma noktası

**Anti-pattern:** Paylaşılan ara durum kullanan görevler için fan-out. Sıralı bağımlılık varsa → pipeline.

---

### Pattern 4: Multi-Agent Debate

**Mimarisi:**
```
[Agent A] ←──── ortak konuşma ────► [Agent B]
     │              │                    │
     └──────────────►  [Judge Agent]  ◄──┘
                         (arbitration)
```

**Ne zaman kullanılır:**
- Yüksek stake kararlar (dış görünürlüklü, geri alınamaz)
- Hallucination'ı minimize etme kritik
- Araştırma sentezi

**Gerçek örnek:** Microsoft Copilot Council — GPT-5.4 + Claude paralel çalışır, judge model arabuluculuk yapar.

**Avantajları:** Agent'lar birbirinin hatalarını yakalar. Tek model sorgusuna kıyasla hallucination azalır.

**Dezavantajları:**
- ~2.5× maliyet baseline'a kıyasla
- **Sycophancy cascading**: agent'lar çoğunluk görüşünü güçlendirir, yanlış consensus üretir
- Sonsuz döngü riski → hard max round count zorunlu (Microsoft: ≤3 agent)

**Anti-pattern:** Rutin Q&A veya düşük stake içerik için debate. Gereksiz %150 overhead.

---

### Pattern 5: Dynamic Handoff (Peer-to-Peer)

**Mimarisi:**
```
[Agent A] ──transfer──► [Agent B] ──transfer──► [Agent C]
   (runtime kararı)        (runtime kararı)
```

**Ne zaman kullanılır:**
- Gereken uzmanlık başlangıçta bilinemiyor
- Müşteri desteği (billing sorunu → teknik köken çıkabilir)
- Dinamik routing

**Gerçek örnek:** HCLTech — dinamik handoff ile %40 daha hızlı vaka çözümü

**Dezavantajları:**
- **En önemli failure mode:** Sonsuz döngü — agent'lar birbirini sorumluluk almadan yönlendirir
- Non-deterministik: aynı input farklı agent zinciri üretebilir → debug çok zor
- Context ya tam geçirilir (pahalı, window sınırlı) ya özetlenir (kayıplı)
- Düzenlenmiş sektörler için genellikle savunulamaz mimari

**Anti-pattern:** Öngörülebilir iş akışları için. Görev yapısı biliniyorsa → supervisor.

---

### Pattern 6: Adaptive Planning

**Mimarisi:**
```
[Manager Agent] ← gözlem → [Planı revize et] ← uzman girdi → [Specialists]
       │
   [Plan discovery in-flight]
```

**Ne zaman kullanılır:**
- Açık uçlu, sabit çözüm yolu olmayan görevler
- Incident response, karmaşık migration
- Kapsam çalışma sırasında değişiyor

**Avantajları:** Doğruluğu hıza tercih eder. Plan kendini düzeltir.

**Dezavantajları:**
- Goal drift: iteratif iyileştirme orijinal hedeften uzaklaşır
- Backtrack edilen dallar compute harcar, tahmin edilemeyen maliyet
- Belirsiz başlangıç talebi → sonsuz döngü

---

## Claude Code'a Özgü — Agent Teams

**Nasıl etkinleştirilir:**
```json
// settings.json
{ "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }
```

**Subagent vs Agent Teams farkı:**

| | Subagent | Agent Team |
|---|---|---|
| İletişim | Sadece parent'a raporlar | Birbirleriyle doğrudan mesajlaşabilir |
| Context | Parent'ın context'i içinde | Ayrı context window |
| Koordinasyon | Parent üzerinden | Shared task list + Mailbox |
| Kullanım | Odaklı, tek dönen sonuç | Paralel, koordineli iş |

**Gerçek dünya:** Anthropic'in Claude Code Review (Mart 2026) — her PR'da agent team deploy eder. Internal code review coverage %16 → %54.

**16 paralel opus agent:** 2.000 Claude Code session'da 100.000 satırlık Rust C derleyicisi (Linux kernel'i derleyebiliyor).

---

## Pattern Seçim Karar Ağacı

```
Görev multi-agent gerektiriyor mu?
│
├── Hayır → Single agent (daha ucuz, 64% vakada eşdeğer)
│
└── Evet
    │
    ├── Alt görev yapısı baştan belli mi?
    │   ├── Evet → Supervisor/Worker (default 2026)
    │   └── Hayır → Adaptive Planning veya Dynamic Handoff
    │
    ├── Görevler bağımsız mı?
    │   ├── Evet, 4+ bağımsız görev → Fan-Out
    │   ├── Evet ama sıralı → Pipeline
    │   └── Hayır, birbirini etkiliyor → Supervisor
    │
    ├── Doğrulama/kalite kritik mi?
    │   ├── Evet, çok yüksek stake → Debate (2.5× maliyet kabul)
    │   └── Hayır → Supervisor + evaluator-optimizer loop
    │
    └── 50+ eşzamanlı bağımsız görev?
        ├── Evet → Swarm (custom engineering gerekir)
        └── Hayır → Supervisor + fan-out branches
```

---

## Kombinasyon Pattern'ları (Üretim Gerçeği)

Hiçbir üretim sistemi tek pattern kullanmaz:

**En yaygın hibrit (3-10 worker):**
```
Supervisor + Fan-Out branches
└── Supervisor decomposes
    └── Fan-out for independent subtasks
    └── Pipeline for ordered subtasks
    └── Debate for critical decisions only
```

**Progressif ölçekleme:**
1. Supervisor ile başla
2. Bağımsız alt görevler için fan-out ekle
3. Yüksek stake kararlar için debate ekle
4. 50+ concurrent agent gerekirse swarm'a geç

---

## Bu Proje İçin Seçim

**Claude Workers = Supervisor/Worker pattern (basit versiyon)**

```
Manager Script (Supervisor)
├── SQLite task queue
├── Worker spawn (claude -p subprocess)
├── Result collection (result.json)
└── Stale recovery

Claude Worker (Worker)
├── Tek görevi alır
├── Bağımsız çalışır
├── Result yazar
└── Exit
```

**Neden Supervisor/Worker:**
- %70 üretim payı — en iyi anlaşılmış pattern
- Native framework desteği en geniş
- Bu proje için peer-to-peer koordinasyon gerekmiyor
- Agent Teams (experimental) yerine kendi implementasyonumuz → daha kontrollü

**İleride eklenebilir:**
- Fan-out: birden fazla worker'ı paralel spawn et (zaten planlandı)
- Debate: kritik görevler için iki worker + evaluator
- Pipeline: görevler arası bağımlılık gerekirse