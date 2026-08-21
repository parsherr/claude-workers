# researches.md — Research Sistemi

> **Bağlantılar:** [[CLAUDE]] | [[hooks]] | [[work-management-system-session-notes]]

Bu dosya, claude-workers projesindeki tüm araştırma sürecini yönetir.
Deep-research yapacak her ajan önce bu dosyayı okumalı, kuralları öğrenmeli, sonra çalışmalı.

---

## Klasör Yapısı

```
claude-workers/
└── researches/
    ├── 001-worker-agent-patterns.md
    ├── 002-task-queue-systems.md
    ├── 003-claude-code-subprocess.md
    └── ...
```

Her research dosyası: `NNN-kısa-slug.md` formatında numaralı.
Numara sırası korunur — yeni research bir sonraki numarayı alır.

---

## Research Dosyası Formatı

Her `.md` dosyası şu şablonu izler:

```markdown
# NNN — Başlık

> **Bağlantılar:** [[researches]] | [[CLAUDE]] | [[ilgili-research]]

**Tarih:** YYYY-MM-DD
**Durum:** draft | complete | archived
**Kaynak türü:** web | paper | codebase | experiment | conversation

---

## Özet (1-3 cümle)

...

## Bulgular

...

## Proje İçin Anlamı

Bu research şu soruya yanıt verir: ...
Bu bulgu şu fikri destekler/çürütür: ...

## Referanslar

- [Kaynak adı](URL)
- [[ilgili-research-dosyası]]

## Açık Sorular

- [ ] Henüz araştırılmamış yan soru
- [ ] Denenmesi gereken yaklaşım
```

---

## Research Listesi

| # | Slug | Başlık | Durum | Tarih |
|---|------|--------|-------|-------|
| 001 | [[researches/001-iyi-agent-yazmanin-kurallari]] | İyi Agent Yazmanın Kuralları | complete | 2026-08-21 |
| 002 | [[researches/002-ornek-agentlar-pattern-analizi]] | Örnek Agent'lar: Pattern ve Anti-Pattern Analizi | complete | 2026-08-21 |
| 003 | [[researches/003-claude-code-headless-spawn-mekanizmasi]] | Claude Code Subprocess/Headless Spawn Mekanizması | complete | 2026-08-21 |
| 004 | [[researches/004-worker-manager-ipc-iletisim]] | Worker-Manager İletişim Katmanı (IPC Seçenekleri) | complete | 2026-08-21 |
| 005 | [[researches/005-task-queue-persistence]] | Task Queue Persistence (SQLite vs Dosya Tasarımı) | complete | 2026-08-21 |
| 006 | [[researches/006-multi-agent-koordinasyon-pattern]] | Multi-Agent Koordinasyon Pattern'ları | complete | 2026-08-21 |
| — | agent-generator/agent-generator.md | Agent Generator Agent yazıldı (Adım 4) | complete | 2026-08-21 |

*(Her yeni research tamamlandığında bu tablo güncellenir.)*

---

## İş Süreci — Research Nasıl Yapılır?

### 1. Konu Belirleme
Proje içinde bir soru ortaya çıkar ("Worker nasıl spawn edilir?", "Hangi IPC mekanizması?").
CLAUDE.md veya mevcut notlardan bağlam okunur. Bu dosyadaki **Açık Sorular** bölümü kontrol edilir.

### 2. Research Açma
`researches/` klasöründe yeni numaralı dosya oluşturulur:
```bash
# Mevcut en yüksek numarayı bul, +1 yap
ls researches/ | sort | tail -1
# → 002-task-queue-systems.md  ⇒  yeni dosya: 003-...
```

Şablon doldurulur, `Durum: draft` ile başlanır.

### 3. Araştırma
- Web araştırması: `WebSearch` + `WebFetch` araçları
- Codebase araştırması: `Glob` + `Read` + `Bash`
- Deneysel: küçük kod snippet'i yazıp test et
- Mevcut researchlere cross-reference: `[[NNN-slug]]` ile bağla

### 4. Bulguları Kaydet
Research dosyasını **Bulgular** ve **Proje İçin Anlamı** bölümleriyle doldur.
Her bulgu için kaynak göster. Belirsiz kalan şeyleri **Açık Sorular**'a ekle.

### 5. Research Tamamlama
- `Durum: complete` yap
- Bu dosyadaki (researches.md) **Research Listesi** tablosunu güncelle
- İlgili proje notlarına `[[NNN-slug]]` ile link ekle
- CLAUDE.md'nin ilgili bölümüne özet bilgiyi yansıt (gerekirse)

### 6. Fikirden Fikre — Sentez
Research'ler birikince sentez yapılır:

```
Research A + Research B → Yeni Fikir / Karar
```

Bunun için:
- İlgili research'leri yan yana oku
- Ortak noktaları ve çelişkileri bul
- Yeni bir **sentez notu** yaz (`researches/NNN-sentez-konusu.md`)
- Sentez notunu kaynak research'lere `[[link]]` ile bağla

---

## Referanstan Fikir Üretme — Kural

Bir research'e dayanarak yeni fikir üretildiğinde:
1. Yeni fikir nerede (hangi dosyada) not edilirse orada `[[NNN-slug]]` ile kaynak gösterilir
2. Research dosyasında da bu fikre ters link eklenir (Açık Sorular veya yeni bir "Bu Research'ten Doğan Fikirler" bölümü)
3. Böylece Obsidian graph'ta: `fikir ↔ research ↔ kaynak` zinciri görünür

---

## Açık Araştırma Soruları (Backlog)

- [ ] Claude Code subprocess olarak nasıl spawn edilir? (headless mod var mı?)
- [ ] Worker ↔ Manager arası en hafif IPC mekanizması nedir? (pipe / unix socket / dosya?)
- [ ] Task queue için en basit persistence yöntemi nedir? (SQLite? JSON dosyası? Redis?)
- [ ] check-completion.py hook'u worker context'inde nasıl davranır?
- [ ] Paralel worker'lar aynı dosyaya yazarsa race condition nasıl önlenir?
- [ ] Worker timeout / watchdog mekanizması nasıl tasarlanır?

---

## Deep Research Agent Talimatları

Bu projeye deep research yapacak her ajan şu adımları izler:

1. **Önce oku:** `CLAUDE.md` → projenin amacını ve mevcut durumunu öğren
2. **Sonra oku:** `researches.md` (bu dosya) → research sistemini ve açık soruları öğren
3. **Mevcut research'leri tara:** `researches/` klasöründeki dosyaları listele, ilgili olanları oku
4. **Yeni research aç:** Şablona uygun yeni dosya oluştur
5. **Araştır ve kaydet:** Bulgularını doldur, kaynak göster
6. **Güncelle:** Bu dosyadaki tabloyu ve ilgili notları güncelle
7. **Bağla:** `[[link]]` ile ilgili dosyalara bağlan

---

> **Not:** Her research bir soruyu yanıtlar. Her yanıt yeni sorular doğurur. Backlog canlı tutulur.