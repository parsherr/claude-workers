> **Bağlantılar:** [[CLAUDE]] | [[agent-generator/core-idea]] | [[researches/001-iyi-agent-yazmanin-kurallari]]

# Agent Yazmanın Kuralları

*Bu dosya Research 001'den çıkarılan best practice listesidir. Agent Generator bu listeyi kullanır.*

**Kaynak:** [[researches/001-iyi-agent-yazmanin-kurallari]] | Tarih: 2026-08-21

---

## Bölüm 1 — System Prompt Kuralları

### SP-1: Runbook yaz, kişilik taslağı değil
System prompt bir "otonom süreç talimatnamesi"dir. Şunları içermeli:
- Rol ve kapsam (ne yapıyor, kimin adına, nerede çalışıyor)
- **Başarı kriterleri** — "bitti" nasıl görünür, ölçülebilir mi?
- **Eskalasyon koşulları** — ne zaman durulur, ne zaman insan istenir?
- Sınırlar ve yasaklar — agent neyi yapmamalı?
- Beklenen output formatı

### SP-2: Tool hiyerarşisini açıkça yaz
Araçların öncelik sırasını söyle. Örnek:
```
Tool kullanım sırası:
1. Önce local cache'e bak
2. Cache yoksa dosya sistemini oku
3. Dosya sisteminde yoksa web araması yap
```

### SP-3: Completion kriterini tanımla
"Görev tamamlandığında şunlar olmuş olmalı: X, Y, Z" şeklinde somut kriterler listele.

### SP-4: Claude için XML tag kullan
Claude, XML tag'leri ve explicit düşünme blokları (`<thinking>`) ile en iyi sonucu verir.

### SP-5: Negatif değil pozitif yönlendir
"Bunu yapma" yerine "Şunu yap" tercih et. Yasaklar gerekliyse kısa ve net tut.

### SP-6: Context'i bütçe gibi yönet
System prompt'a sadece her çağrıda gerçekten gerekli olan şeyi koy. Büyük dökümanları lazy-load et (sadece ihtiyaç duyulduğunda yükle).

---

## Bölüm 2 — Tool Seçimi ve Tasarımı

### T-1: Az ama keskin tool
Çakışan sorumluluğu olan büyük toolset < net sınırlı az sayıda tool.
Toolset genişledikçe agent kafa karışıklığı artar.

### T-2: Her tool bir docstring gibi belgelenecek
Her tool tanımı şunları içermeli:
- Ne yapar (tek cümle)
- Hangi durumda kullanılır
- Örnek input
- Edge case'ler ve sınırlamalar
- Benzer tool'lardan farkı

### T-3: Poka-yoke argüman tasarımı
Hata yapmayı zorlaştıracak şekilde tasarla:
- Relative path → Absolute path (Claude Code SWE-bench dersi)
- Belirsiz string → Enum/sabit değer
- Opsiyonel ama kritik parametre → Required

### T-4: İzin verilmeyen tool'ları gösterme
"Deny-first" ilkesi: agent izin verilmeyen araçları görmemeli bile. Tool pre-filtering harness'ta yapılır, prompt'ta değil.

### T-5: Tool hata mesajları LLM'e anlamlı olmalı
Tool bir hata döndürdüğünde mesaj şunları içermeli:
- Ne başarısız oldu
- Muhtemel sebep
- Agent'ın yapabileceği alternatif

---

## Bölüm 3 — Completion (İşi Bitirme)

### C-1: Stop koşullarını harness'ta tanımla
5 temel stop koşulu (Claude Code'dan):
1. **Tool kullanımı yok** → metin-only çıktı = birincil stop sinyali
2. **maxTurns** limiti aşıldı
3. **Context overflow** (tüm recovery sonrası)
4. **Hook müdahalesi** (PostToolUse hook sinyal gönderdi)
5. **Explicit abort**

### C-2: "Başarı gibi görünen başarısızlık"tan kork
En tehlikeli hatalar 200 OK döner ama yanlış iş yapar. Completion detection için:
- Deterministik check: output formatı doğru mu?
- Semantic check: task gerçekten tamamlandı mı? (LLM-judge)
- Trace inspection: beklenmedik loop var mı?

### C-3: Completion'ı 3 katmanda doğrula
1. **Schema/format:** Beklenen formatta çıktı üretildi mi?
2. **Semantic:** Kullanıcının hedefi gerçekten karşılandı mı?
3. **Side effect:** İstenen yan etkiler gerçekleşti mi? (dosya yazıldı, API çağrıldı vs.)

### C-4: Budget guardrail koy
Her agent için maksimum limit belirle:
- **Turn limit:** Tipik agent 5-15 adım. Hard ceiling: 25-30.
- **Token budget:** Aşılırsa partial result döndür, logla.
- **Wall-clock timeout:** Beklenen sürenin 3-5x'i → anormal durum.

### C-5: Budget aşılınca partial result döndür
Zorla durduğunda:
- Tamamlananları raporla
- Nerede durduğunu söyle
- Neden durduğunu söyle
- Devam etmek için ne gerektiğini söyle

---

## Bölüm 4 — Hata Yönetimi

### E-1: Validation gate — tool çağrısından ÖNCE
Her tool çağrısından önce 3 katman doğrulama:
1. **Schema:** Input formatı doğru mu?
2. **Business logic:** Değer mantıklı mı? Entity var mı?
3. **Safety/Policy:** İzin var mı? PII riski var mı?

### E-2: Idempotent saga — multi-step workflow zorunluluğu
Aynı adımın iki kez çalışması güvenli olmalı (idempotency):
- Her adımdan önce intent'i durable store'a yaz
- Idempotency key: `workflow_id + step_number`
- Her adımın geri alma (compensation) aksiyonu olmalı

### E-3: Retry stratejisi
```
wait = min(base_delay * 2^attempt + random_jitter, max_delay)
```
- Base: 1-2s, Max: 60-120s, Max attempt: 5-7
- Retry: 429, 500-504, timeout
- Fail fast: 400, 401, 403

### E-4: Circuit breaker — HTTP + semantic
Sadece HTTP error'da değil, kalite düştüğünde de tetikle:
- Schema validation fail > %15 → breaker aç
- 5 ardışık hata → breaker aç
- Çıktı kalitesi düşükse → fallback'e geç

### E-5: Hataların compounding'ine dikkat et
Erken adımda küçük bir hata, downstream adımlarda büyük hataya dönüşür. Strateji:
- Her adımda sonucu doğrula (sadece sonda değil)
- Kritik adımlar arasında checkpoint koy
- "Şimdiye kadar ne tamamlandı" durumunu durable store'da tut

### E-6: Silent failure'dan kork
`error` kodu yoksa bile başarısız olunabilir. Özellikle dikkat:
- Boş ama "başarılı" response
- Kısmi ama "tam" görünen sonuç
- Yanlış ama "mantıklı" yanıt

### E-7: Insan eskalasyonu tasarla
Eskalasyon failure değil, bir tasarım özelliği. Şu koşullarda:
- 3 retry sonrası aynı hata
- Safety/policy ihlali
- Geri alınamaz yüksek değerli aksiyon
- Kullanıcı açıkça istedi

---

## Bölüm 5 — Mimari İlkeler

### A-1: Harness > Model
Claude Code'un dersi: %1.6 AI mantığı, %98.4 operasyonel harness. Model yetenekleri yakınsadıkça **güvenilirliği belirleyen altyapıdır**. Worker queue, state machine, monitoring, retry — bunlara yatırım yap.

### A-2: Basitlikle başla
Anthropic'in tavsiyesi: En basit çözümle başla. Karmaşıklık "ölçülebilir şekilde iyileştirdiğinde" ekle. Birçok görev iyi optimize edilmiş tek bir LLM çağrısıyla çözülür.

### A-3: Reversibility-weighted risk
- Read-only işlemler → paralel çalıştır
- State-modifying işlemler → serialize et
- Geri alınamaz işlemler → insan onayı iste

### A-4: Context izolasyonu
Subagent başarısızlıkları veya context şişmesi parent session'ı kirletmemeli. Her worker'ın kendi context window'u var.

### A-5: Deny-first permission
Tanınmayan aksiyonlar varsayılan olarak bloklanır. İzin vermek aktif bir karar; izin vermemek default.

### A-6: Şeffaf konfigürasyon
Agent talimatları ve belleği, version-controllable dosyalarda (CLAUDE.md gibi) tutulur. Opak database veya embedding'e gömme.

---

## Bölüm 6 — Akıl Yürütme Döngüsü (ReAct + Reflection)

### R-1: ReAct döngüsünü uygula
```
Düşün → Tool çağır → Sonucu gözlemle → Tekrar düşün → ...
```
Her adımda "bu sonuç bana ne söylüyor ve sıradaki adım ne olmalı?" sorusunu sor.

### R-2: Reflection ekle
Karmaşık görevlerde üret → eleştir → iyileştir döngüsü:
- Çıktını üret
- "Bu gerçekten soruyu yanıtlıyor mu?" diye eleştir
- Eksikleri gidererek iyileştir

### R-3: Katmanlı pattern kullan
Üretim sistemleri tek pattern kullanmaz:
```
Plan → ReAct → Reflection → Verify
```

---

## Bölüm 7 — Multi-Agent Koordinasyonu

### M-1: Her agent'ın kapsamını dar tut
Geniş scope < dar scope. Agent'lar birbiriyle scope çakışırsa koordinasyon bozulur.

### M-2: Subagent sadece özet döndürür
Subagent tüm context'ini parent'a aktarmamalı — sadece özet. Context inflation'ı önler.

### M-3: Orchestrator-Worker ayrımı
Orchestrator: planlar, dağıtır, sentezler.
Worker: tek bir iyi tanımlanmış görevi çalıştırır, sonucu raporlar.

### M-4: Shared prompt kullanma
Her agent için ayrı, hedefe özel system prompt yaz. Paylaşılan prompt → scope creep → kafa karışıklığı.

### M-5: Worker idempotent olmalı
Aynı görev iki worker tarafından çalıştırılırsa güvenli olmalı (work stealing senaryosu).

---

## Özet Kontrol Listesi

Yeni bir agent yazarken bu listeyi tara:

**System Prompt**
- [ ] Rol ve kapsam net mi?
- [ ] Başarı kriterleri somut ve ölçülebilir mi?
- [ ] Tool hiyerarşisi açıkça yazılı mı?
- [ ] Eskalasyon koşulları tanımlanmış mı?
- [ ] Output formatı belirtilmiş mi?

**Tool Tasarımı**
- [ ] Her tool belgelenmiş mi (docstring kalitesinde)?
- [ ] Poka-yoke argüman tasarımı yapıldı mı?
- [ ] İzinsiz tool'lar model'den gizlendi mi?

**Completion**
- [ ] maxTurns / budget guardrail var mı?
- [ ] Semantic completion check var mı?
- [ ] Budget aşılınca partial result döndürüyor mu?

**Hata Yönetimi**
- [ ] Validation gate tool çağrısından ÖNCE çalışıyor mu?
- [ ] Retry stratejisi + exponential backoff var mı?
- [ ] Multi-step workflow idempotent mi?
- [ ] Silent failure koruması var mı?

**Mimari**
- [ ] Harness logic model logic'ten ayrılmış mı?
- [ ] Context izolasyonu sağlandı mı?
- [ ] State durable store'da tutuluyor mu?