> **Bağlantılar:** [[CLAUDE]] | [[kiwimi-workers/the-project-idea]] | [[work-management-system-session-notes]] | [[agent-generator/anatomy-team-lead]] | [[agent-generator/anatomy-worker]] | [[agent-generator/system-design]]

# Agent Generator — Temel Fikir

## Ne?
İyi, yeniden kullanılabilir Claude agent'ları üretmek için bir meta-agent sistemi.
Agent Generator kendi başına bir agent'tır; ona "şu işi yapan bir agent lazım"
dersin, o da sana hazır, optimize edilmiş bir agent dosyası çıkarır.

## Neden?
Her projede benzer agent'lar sıfırdan yazılıyor. İyi agent yazmak zordur:
doğru tool seçimi, doğru system prompt yapısı, doğru completion kriterleri gerektirir.
Bu bilgiyi bir kez biriktirip defalarca kullanmak gerekiyor.

## Nasıl Yapılacak — Adım Adım

### Adım 1: İyi Agent Yazmanın Kurallarını Çıkar
- Araştırma: iyi agent ne demek? (tool seçimi, system prompt, completion, hata yönetimi)
- Çıktı: `agent-generator/rules.md` — best practice listesi

### Adım 2: Örnek Agent'ları İncele
- Mevcut projelerdeki agent'lar analiz edilir
- edinilen tum bilgiler researches klasorunun altinda .md olarak kayit edilir
- Ortak pattern'lar ve anti-pattern'lar belgelenir, bu belgeler agent-generator folderi altinda .md formasi formatinda kayit edilir

### Adım 3: Takım Hâlinde Çalışan Agent'ları Araştır
- Multi-agent coordination: nasıl iletişim kuruyorlar?
- Hangi pattern'lar işe yarıyor? (Supervisor/Worker, Peer-to-peer, Pipeline…)
- Çıktı: `agent-generator/team-patterns.md`

### Adım 4: Agent Generator Agent'ı Yaz
- Input: kullanıcının ihtiyaç tanımı (doğal dil)
- Output: optimize edilmiş `.md` agent dosyası (global scope için)
- Agent; rules.md + team-patterns.md bilgisini kullanarak optimal agent'ı üretir

## Kapsam
**Global scope** — üretilen agent'lar tüm projelerde kullanılabilir,
takım kurguları da dahil (multi-agent team template'leri).