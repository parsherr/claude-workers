# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Bağlantılar:** [[hooks]] | [[work-management-system-session-notes]] | [[researches]] | [[kiwimi-workers/the-project-idea]] | [[agent-generator/core-idea]]

**Her yeni oturumda bu dosyayı oku. Deep research yapacaksan ayrıca [[researches]] dosyasını oku.**

---

## Not Sistemi — Obsidian Graph

**Dil kuralı:** Tüm notlar İngilizce yazılır.

Notlar `[[dosya-adı]]` syntax'ı ile birbirine bağlıdır. Yeni not yazarken ilgili notlara `[[link]]` ekle ve her notun başında bağlantı satırı koy:

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
           CLAUDE ←————————————————————→ CLAUDE
```

---

## Proje Amacı

**Kusursuz bir Claude Code Worker Agent sistemi** inşa etmek.

Bir "worker agent", verilen bir görevi alıp baştan sona eksiksiz tamamlayan, onay beklemeden çalışan, bağımsız bir Claude Code sürecidir. Hedef: insan müdahalesi olmadan güvenilir şekilde iş yapan otonom ajan altyapısı.

```
Kullanıcı → Görev Tanımı → Worker Agent → Tamamlanmış İş
                                |
                     (hiç onay istemeden, tamamen bağımsız)
```

---

## Mevcut Durum (Ağustos 2026)

**Saf planlama + araştırma aşaması — henüz çalıştırılabilir kod yok.**

Çalışma dizini: `/home/dogukan/Documents/github/claude-workers`
Model: `claude-opus-5-0` (Rusk gateway) · Platform: Linux (zsh)

---

## Dosya Haritası

| Dosya / Klasör | İçerik |
|----------------|--------|
| `CLAUDE.md` | Bu dosya — proje rehberi |
| `work-management-system-session-notes.md` | İlk tasarım notları, kavramsal altyapı |
| `hooks.md` | check-completion.py hook analizi |
| `need-to-fix.md` | Bilinen sorunlar ve düzeltme listesi |
| `researches.md` | Research sistemi, kurallar, backlog |
| `researches/` | Numaralı research dosyaları (NNN-slug.md) |
| `kiwimi-workers/the-project-idea.md` | Kiwimi Workers proje fikri |
| `agent-generator/core-idea.md` | Agent Generator fikri |
| `agent-generator/rules.md` | İyi agent yazmanın kuralları |
| `agent-generator/patterns-and-anti-patterns.md` | Agent pattern'leri ve anti-pattern'ler |
| `agent-generator/team-patterns.md` | Çok-ajanlı takım koordinasyon pattern'leri |
| `agent-generator/anatomy-team-lead.md` | Team Lead agent anatomisi (tam şablon) |
| `agent-generator/anatomy-worker.md` | Worker agent anatomisi (tam şablon) |
| `agent-generator/system-design.md` | Spawn + IPC + state machine + Manager Script |
| `~/.claude/hooks/check-completion.py` | Aktif stop hook |
| `~/.claude/settings.json` | Global ayarlar |

---

## Kritik Tasarım Prensipleri

### 1. Worker = Tamamlayan Ajan

Worker agent'lar asla yarım bırakmaz. `check-completion.py` hook'u bunu enforce eder: Claude "Devam edeyim mi?" diye soramaz — ya işi yapar ya hata bildirir. Detay: [[hooks]]

Hook mekanizması: Her `claude` süreci kapanırken hook çalışır → LLM semantic check (Haiku) veya regex fallback → görev bitmemişse `exit 2` ile Claude'u devam ettirir.

### 2. Work Management System

- Merkezi görev kuyruğu: ID, durum (pending/running/done/failed), atanan worker, sonuç
- Worker'lar idempotent olmalı — aynı görev iki kez çalışırsa güvenli
- Detay: [[work-management-system-session-notes]]

### 3. Claude Code as Runtime

Worker'lar ayrı `claude` süreçleri olarak çalışır. Her worker kendi bağlamına sahip; paralel görev işleme desteklenir.

### 4. İki Bağlantılı Alt Proje

**Kiwimi Workers** ([[kiwimi-workers/the-project-idea]]): Spesifik bir worker sistemi uygulaması.

**Agent Generator** ([[agent-generator/core-idea]]): İyi agent `.md` dosyaları üretmek için bir meta-ajan. Biriken bilgi tabanı:
- `rules.md` — İyi agent tanımının 10 kuralı
- `patterns-and-anti-patterns.md` — Agent pattern'leri (coordinator, specialist, reviewer) ve anti-pattern'ler
- `team-patterns.md` — Çok-ajanlı takım koordinasyon pattern'leri (pipeline, hub-and-spoke, vb.)

---

## Mimari

```
+--------------------------------------+
|        Work Management System        |
|  - Task Queue                        |
|  - Worker Registry                   |
|  - Result Store                      |
+------------------+-------------------+
                   | assign task
         +---------v---------+
         |   Worker Agent 1  |  ← Claude Code process
         | (check-completion |
         |  hook aktif)      |
         +-------------------+
         +-------------------+
         |   Worker Agent 2  |  ← Claude Code process
         +-------------------+
```

---

## Research Sistemi

Tüm araştırmalar `researches/` klasöründe `NNN-slug.md` formatında saklanır. Deep research yapacak her ajan şu sırayı izler:

1. Bu dosyayı (CLAUDE.md) oku
2. `researches.md` dosyasını oku — açık soruları öğren
3. `researches/` klasörünü tara — mevcut araştırmaları kontrol et
4. Yeni research dosyası oluştur, araştır, kaydet
5. `researches.md` tablosunu ve ilgili notları güncelle

Detaylı kurallar ve şablon: [[researches]]

---

## Önümüzdeki Adımlar

1. `researches/` klasöründeki açık soruları araştır → [[researches]]
2. Work Management Protocol — görev formatı ve durum makinesi tasarımı
3. Worker spawn mekanizması — yeni `claude` süreci nasıl başlatılır
4. İletişim katmanı — worker ↔ manager arası veri transferi
5. Hata yönetimi — worker çöktüğünde ne olur
6. MVP testi — tek worker, tek görev, uçtan uca akış

---

> **Her yeni Claude oturumunda:** Bu CLAUDE.md'yi oku → araştırma yapacaksan `researches.md`'yi oku → nerede olduğumuzu anla → çalış.