> **Bağlantılar:** [[CLAUDE]] | [[agent-generator/core-idea]] | [[agent-generator/rules]] | [[agent-generator/patterns-and-anti-patterns]] | [[agent-generator/anatomy-team-lead]] | [[agent-generator/anatomy-worker]] | [[agent-generator/system-design]] | [[researches/006-multi-agent-koordinasyon-pattern]]

# Multi-Agent Team Pattern'ları

*Bu dosya Research 006'dan çıkarılan team coordination kataloğudur. Agent Generator bu listeyi kullanır.*

**Kaynak:** [[researches/006-multi-agent-koordinasyon-pattern]] | Tarih: 2026-08-21

---

## Önce: Multi-Agent Gerekli mi?

**Kritik ekonomi testi — bu soruyu sor:**

```
Görev şunlardan en az birinden gerçekten yararlanıyor mu?
□ Paralelizm: bağımsız alt görevler aynı anda çalışabilir
□ Uzmanlaşma: farklı domain expertise gerektiren ayrı alanlar
□ Eleştiri: agent'ların birbirinin hatalarını yakalaması gerekiyor
```

Hayırsa → **single agent kullan**. Princeton NLP 2026: eşdeğer araç ve context'le single agent, görevlerin %64'ünde multi-agent'ı geçiyor veya eşit.

**Token overhead gerçeği:**
- Supervisor/Worker: +%285 overhead
- Peer-to-peer: +%58 overhead
- Debate: +%150 overhead (2.5× maliyet)

---

## Pattern Kataloğu

### TP-01: Supervisor/Worker (2026 Üretim Standardı)

**Yapı:**
```
        [Supervisor]
        /     |     \
  [W-1]   [W-2]   [W-3]
```

**Ne zaman:** Görev farklı uzmanlık alanları gerektiriyor + alt görev yapısı önceden bililebilir. Üretim payı: ~%70.

**Supervisor sorumlulukları:** Planlama → alt görevlere bölme → worker'lara dağıtma → sonuçları sentezleme → kalite kontrolü.

**Worker sorumlulukları:** Tek, dar tanımlı görevi eksiksiz tamamla → sonucu raporla → dur.

**Model seçimi:** Supervisor = opus (karar verici). Worker = sonnet veya haiku (görev tipine göre).

**Kritik kurallar:**
- Supervisor her zaman tek — iki supervisor = koordinasyon çakışması
- Worker'lar birbirini görmez, sadece supervisor'a raporlar
- 4+ worker → context overflow riski → batch veya pipeline kombinasyonu
- Over-delegation anti-pattern: alt görevler çok küçük → worker'lar eksik döndürür → iteration döngüsü

**Bu proje versiyonu:**
```
Manager Script → SQLite queue → claude -p worker subprocess
Worker sonucu: result.json → Manager okur → SQLite günceller
```

---

### TP-02: Sequential Pipeline

**Yapı:**
```
[A] → çıktı → [B] → çıktı → [C] → Sonuç
```

**Ne zaman:** Katı sıralı bağımlılıklar — her adım öncekinin tam çıktısını gerektirir.

**İdeal kullanım:** Araştır → taslak yaz → eleştir → düzelt → doğrula.

**Kritik kural:** Her aşamada çıktıyı doğrula. İlk aşama hatası cascade failure üretir — sonraki tüm aşamalar kirletilmiş input üzerinde çalışır.

**Kaçınılacak anti-pattern:** Paralel çalışabilecek aşamaları pipeline'a sokmak → gereksiz latency (toplam = tüm aşamaların süresi).

**4 agent pipeline gerçekliği:** ~950ms koordinasyon overhead, ~500ms gerçek işlem. Yüksek maliyet.

---

### TP-03: Fan-Out / Fan-In (Paralel)

**Yapı:**
```
[Dispatcher]
  ├──► [A] ─┐
  ├──► [B] ─┼──► [Aggregator] → Sonuç
  └──► [C] ─┘
```

**Ne zaman:** 4+ bağımsız alt görev, bağımsız araştırma kaynakları, multi-perspektif analiz.

**Avantaj:** Duvar saati = en yavaş branch (toplamı değil). ~%75 latency azalması.

**Kritik kurallar:**
- Partial failure kararını önceden al: bir branch hata → tümünü durdur mu / kısmi döndür mu / retry mi?
- Shared state race condition: N×(N-1)/2 çakışma noktası — agent'lar arası paylaşılan state varsa fan-out yerine supervisor kullan
- Rate limit: 15 paralel agent bireysel limitleri geçmese de paylaşılan kapasiteyi aşabilir → batch spawn + gecikme

**Anti-pattern:** Görevler arası bağımlılık varken fan-out. Bağımlılık varsa → supervisor veya pipeline.

---

### TP-04: Multi-Agent Debate

**Yapı:**
```
[Agent A] ──►
              [Judge] → Karar
[Agent B] ──►
```

**Ne zaman:** Yüksek stake, geri alınamaz kararlar. Hallucination'ın minimize edilmesi kritik. Araştırma sentezi.

**Basit versiyon (Maker-Checker):**
```
[Maker] → üretir → [Checker] → onaylar/reddeder → tekrar
```
- Maker: ucuz hızlı model (haiku/sonnet)
- Checker: güçlü model (opus)
- ~%20 ek maliyet single call'a kıyasla

**Tam debate:** 2.5× maliyet. Yalnızca gerçekten yüksek stake için.

**Kritik kurallar:**
- Hard max round count zorunlu — yoksa sonsuz döngü
- Microsoft önerisi: ≤3 agent
- Sycophancy cascading riski: agent'lar çoğunluk görüşünü güçlendirir → yanlış consensus
- Judge model bias: judge bir modelin stilini tercih edebilir → doğruluğu değil stili ödüllendirme

**Anti-pattern:** Rutin görevler için debate. Yalnızca gerçekten yüksek stake: hukuki karar, kritik güvenlik kararı, geri alınamaz aksiyon.

---

### TP-05: Dynamic Handoff (Peer-to-Peer)

**Yapı:**
```
[A] → "B daha uygun" → [B] → "C gerekiyor" → [C] → Sonuç
      (runtime kararı)        (runtime kararı)
```

**Ne zaman:** Gereken uzmanlık başlangıçta bilinemiyor. Müşteri destek triage. Dinamik routing.

**Kritik uyarı:** En riskli pattern. #1 failure mode: agent'lar sonsuz döngüde birbirini yönlendirir.

**Kurallar:**
- Hard ownership timeout: N transfer sonrası son agent mecburen sahiplenir
- Context geçişi planla: tam geçiş (pahalı) veya özetleme (kayıplı) — birini seç
- Düzenlenmiş sektörler (finans, sağlık) için genellikle audit requirement'ları karşılanmaz

**Bu proje için: KULLANMA.** Supervisor/Worker ile başla.

---

### TP-06: Adaptive Planning

**Yapı:**
```
[Manager] ←─ gözlem ─► [Plan revizesi] ←─ uzman girdi ─► [Uzmanlar]
    └── Plan çalışma sırasında keşfedilir
```

**Ne zaman:** Açık uçlu, kapsam değişen görevler. Incident response. Karmaşık migration.

**Kritik uyarı:** Goal drift — iteratif iyileştirme orijinal hedeften uzaklaşır. Maliyet tavanı yok.

**Bu proje için: ŞİMDİLİK KULLANMA.** MVP'de Supervisor/Worker yeterli.

---

## Kombinasyon Kılavuzu

**Üretim gerçeği:** Tek pattern kullanılmaz. Katmanlanır.

### Önerilen progressif ölçekleme:

**Seviye 1 — MVP (bu proje):**
```
Supervisor + single worker at a time
```

**Seviye 2 — Paralel workers:**
```
Supervisor + Fan-Out (3-10 bağımsız görev aynı anda)
```

**Seviye 3 — Kalite katmanı:**
```
Supervisor + Fan-Out + Maker-Checker (kritik görevler için)
```

**Seviye 4 — Pipeline ihtiyacı:**
```
Supervisor + Pipeline (bağımlı alt görevler zinciri)
```

---

## Bu Proje İçin Karar

**Seçim: Supervisor/Worker (TP-01) — basit versiyonuyla başla**

| Bileşen | Rol | Model |
|---------|-----|-------|
| Manager script | Supervisor | (script — LLM değil) |
| claude subprocess | Worker | sonnet (varsayılan) |

**Neden bu seçim:**
- %70 üretim payı — en iyi anlaşılmış pattern
- Peer-to-peer koordinasyona gerek yok — worker'lar birbirini bilmesine gerek yok
- Experimental Agent Teams yerine kendi kontrolümüzdeki implementasyon
- Fan-out zaten mimariyle destekleniyor (birden fazla parallel worker)
- Gelecekte Debate eklenebilir (kritik görevler için Maker-Checker)

**Olmayacak (şimdilik):**
- Dynamic Handoff: sonsuz döngü riski + debug zorluğu
- Adaptive Planning: kapsam creep + tahmin edilemez maliyet
- Swarm: 50+ concurrent agent olmayacak MVP'de

---

## Claude Code Agent Teams (Referans)

Anthropic'in experimental native multi-agent sistemi. Bizim implementasyonumuzdan farklı:

| Özellik | Agent Teams | Bizim implementasyonumuz |
|---------|------------|--------------------------|
| Etkinleştirme | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | Kendi script'imiz |
| İletişim | File-based inbox polling | result.json + SQLite |
| Worker-Worker iletişim | Destekli (SendMessage) | Yok (MVP'de gereksiz) |
| Durum | Experimental | Production-ready hedef |

Agent Teams stabil hale gelirse geçiş değerlendirilebilir. Şimdilik kendi implementasyon.