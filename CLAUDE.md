# Claude Workers — Proje Rehberi

> **Bağlantılar:** [[hooks]] | [[work-management-system-session-notes]] | [[researches]] | [[kiwimi-workers/the-project-idea]] | [[agent-generator/core-idea]]

Bu dosya, her yeni Claude Code oturumunun projeyi anlaması için yazılmıştır.
**Her oturumda bu dosyayı oku. Deep research yapacaksan ayrıca [[researches]] dosyasını oku.**

---

## Not Sistemi — Obsidian Graph

Bu proje notları **Obsidian vault** olarak organize edilmiştir. Notlar `[[dosya-adı]]` syntax'ı ile birbirine bağlıdır.

- `[[hooks]]` → `hooks.md` — check-completion.py hook analizi
- `[[CLAUDE]]` → bu dosya (hub)
- `[[work-management-system-session-notes]]` → ilk tasarım notları
- `[[researches]]` → `researches.md` — research sistemi ve listesi
- `[[researches/NNN-slug]]` → tek bir research dosyası

**Kural:** Yeni not yazarken ilgili notlara `[[link]]` ekle. Her notun başında:
```
> **Bağlantılar:** [[CLAUDE]] | [[ilgili-not]]
```

Mevcut graf:
```
hooks ←→ CLAUDE ←→ work-management-system-session-notes
           ↕
        researches
           ↕
   researches/NNN-slug (her research dosyası)

kiwimi-workers/the-project-idea ←→ agent-generator/core-idea
              ↕                              ↕
            CLAUDE ←———————————————————→ CLAUDE
```

---

## Research Sistemi

Bu proje araştırma odaklı bir planlama sürecindedir. Tüm araştırmalar `researches/` klasöründe numaralı `.md` dosyaları olarak saklanır.

**Deep research yapacak her ajan şu sırayı izler:**

1. Bu dosyayı (CLAUDE.md) oku — projenin amacını öğren
2. `researches.md` dosyasını oku — research sistemini ve açık soruları öğren
3. `researches/` klasörünü tara — mevcut araştırmaları kontrol et
4. Yeni research dosyası oluştur, araştır, kaydet
5. `researches.md` tablosunu ve ilgili notları güncelle

Detaylı kurallar, şablon ve iş süreci: [[researches]]

---

## Proje Amacı

**Kusursuz bir Claude Code Worker Agent sistemi** inşa etmek.

Bir "worker agent", verilen bir görevi alıp baştan sona eksiksiz tamamlayan, onay beklemeden çalışan, bağımsız bir Claude Code sürecidir. Hedef: insan müdahalesi olmadan güvenilir şekilde iş yapan otonom ajan altyapısı.

---

## Nihai Hedef

```
Kullanıcı → Görev Tanımı → Worker Agent → Tamamlanmış İş
                                |
                     (hiç onay istemeden, tamamen bağımsız)
```

- Görevler queue'ya girer
- Worker agent'lar görevleri alıp çalıştırır
- Her görev ya başarıyla tamamlanır ya da hata raporuyla kapanır
- Çoklu worker desteği (paralel çalışma)
- Work stealing / görev dağıtımı

---

## Mevcut Durum (Ağustos 2026)

**Saf planlama + araştırma aşaması.** Henüz kod yazılmadı.

Elimizde olan:
- `work-management-system-session-notes.md` — İlk tasarım notları, kavramsal altyapı
- `hooks.md` — Mevcut `check-completion.py` hook analizi
- `researches.md` — Research sistemi (yeni kuruldu, backlog dolu)
- `researches/` — Research dosyaları klasörü (henüz boş)
- `~/.claude/hooks/check-completion.py` — Stop hook: Claude'u "yarım bırakma" moduna sokar
- `kiwimi-workers/the-project-idea.md` — Kiwimi Workers proje fikri (birbiriyle bağlantılı)
- `agent-generator/core-idea.md` — Agent Generator core fikri (birbiriyle bağlantılı)

---

## Kritik Tasarım Prensipleri

### 1. Worker = Tamamlayan Ajan
Worker agent'lar asla yarım bırakmaz. `check-completion.py` hook'u bu ilkeyi enforce eder:
Claude "Devam edeyim mi?" diye soramaz — ya işi yapar ya hata bildirir.
Detay: [[hooks]]

### 2. Work Management System
- Görevler merkezi bir sistemde yönetilir
- Her görev: ID, durum (pending/running/done/failed), atanan worker, sonuç
- Worker'lar idempotent olmalı (aynı görev iki kez çalışırsa güvenli)

### 3. Claude Code as Runtime
Worker'lar ayrı Claude Code süreçleri olarak çalışır:
- Her worker'ın kendi bağlamı var (context isolation)
- Paralel görev işleme mümkün
- Claude Code'un araçları (Bash, Read, Write, WebSearch…) worker'lara hazır

### 4. Hook-Driven Completion
`check-completion.py` stop hook'u sistemin kalbidir. Her worker oturumunda:
- LLM semantic check (Haiku model) veya regex fallback ile
- "Gerçekten bitti mi?" sorusunu yanıtlar
- Yanıt "hayır" ise `exit 2` ile Claude'u devam ettirtir

---

## Mimari Fikir

```
+--------------------------------------+
|        Work Management System        |
|  - Task Queue                        |
|  - Worker Registry                   |
|  - Result Store                      |
+------------------+-------------------+
                   | assign task
         +---------v---------+
         |   Worker Agent 1  |  <- Claude Code process
         | (check-completion |
         |  hook aktif)      |
         +-------------------+
         +-------------------+
         |   Worker Agent 2  |  <- Claude Code process
         +-------------------+
```

Detaylar için: [[work-management-system-session-notes]]

---

## Önümüzdeki Adımlar

1. **Araştırma (şimdi):** `researches/` klasöründe açık soruları araştır (bkz. [[researches]])
2. **Work Management Protocol** — görev formatı, durum makinesi tasarımı
3. **Worker spawn mekanizması** — yeni Claude Code süreci nasıl başlatılır
4. **İletişim katmanı** — worker ↔ manager arası veri transferi
5. **Hata yönetimi** — worker çöktüğünde ne olur
6. **MVP testi** — tek worker, tek görev, uçtan uca akış

---

## Proje Dosyaları

| Dosya / Klasör | İçerik |
|----------------|--------|
| `CLAUDE.md` | Bu dosya — proje rehberi (hub) |
| `work-management-system-session-notes.md` | İlk tasarım notları |
| `hooks.md` | check-completion.py hook analizi |
| `researches.md` | Research sistemi, kurallar, backlog |
| `researches/` | Numaralı research dosyaları |
| `kiwimi-workers/the-project-idea.md` | Kiwimi Workers proje fikri |
| `agent-generator/core-idea.md` | Agent Generator core fikri |
| `~/.claude/hooks/check-completion.py` | Aktif stop hook |
| `~/.claude/settings.json` | Global ayarlar (Rusk gateway, opus-5) |

---

## Geliştirme Ortamı

- **Model:** claude-opus-5-0 (Rusk gateway)
- **Platform:** Linux (zsh)
- **Çalışma dizini:** `/home/dogukan/Documents/github/claude-workers`
- **Global hooks:** `~/.claude/hooks/` (check-completion.py aktif)
- **Settings:** `~/.claude/settings.json`

---

> **Her yeni Claude oturumunda:** Bu CLAUDE.md'yi oku → araştırma yapacaksan `researches.md`'yi oku → nerede olduğumuzu anla → çalış.