# hooks.md — check-completion.py Analizi

> **Bağlantılar:** [[CLAUDE]] | [[work-management-system-session-notes]]

> **Konum:** `~/.claude/hooks/check-completion.py`
> **Tip:** Stop Hook — Claude her yanıt verdikten sonra çalışır, durmasına izin verip vermeyeceğine karar verir.

---

## Ne Yapar?

Claude bir yanıt verip durmak istediğinde, bu hook devreye girer ve şunu sorar:
**"Claude gerçekten işi bitirdi mi, yoksa kullanıcıdan onay bekleyerek yarım mı bıraktı?"**

Eğer yarım bıraktıysa → stdout'a `{"decision":"block","reason":"..."}` yazar ve `exit 0` ile çıkar.
Eğer işi bitirdiyse → `exit 0` ile geçer, Claude normal şekilde durur.

---

## Çalışma Mantığı (Adım Adım)

### 1. Input: Stop Hook Event'i Al
```
stdin → JSON event
```
Claude Code, Stop hook'larına bir JSON event gönderir. Event içinde:
- `last_assistant_message`: Claude'un son yanıtı (varsa)
- `transcript`: Tüm konuşma geçmişi (fallback olarak kullanılır)

### 2. Mesajı Çıkar (`get_text`)
`last_assistant_message` varsa onu kullan.
Yoksa `transcript` içinde geriye doğru tararak son `assistant` rolündeki mesajı bul.
İçerik `list` (content block array) ya da `str` olabilir — her ikisini de işler.

### 3. Kısa Mesajları Atla
`len(text) < 30` ise hook `exit 0` ile geçer — çok kısa yanıtlar analiz edilmez.

### 4. LLM Semantic Kontrolü (`llm_check`) — Ana Yol
**Rusk proxy** üzerinden `claude-haiku-4-5-20251001` modeline bir prompt gönderir (curl ile, sync).

**Prompt'un özü:** Claude'a "Bu yanıt tamamlandı mı?" sorusunu sorar.

**Tamamlanmamış (incomplete)** sayılan durumlar:
- İşe başlamak için izin istemek: "Yazayım mı?", "Başlayalım mı?", "Shall I start?"
- Hangi parçadan başlanacağını sormak: "Hangisinden başlayalım?"
- "Hazır olunca söyle", "Onay verirsen" tarzı bekleme
- Kullanıcı onayı olmadan iş yapmayı reddetmek

**Tamamlanmış (complete)** sayılan durumlar:
- Gerçek iş yapılmış (kod yazılmış, analiz edilmiş, düzeltilmiş)
- Gerçekten gerekli bir açıklayıcı soru sorulmuş ("Hangi framework?", "Hangi veritabanı?")
- İş bittikten sonra isteğe bağlı sonraki adımlar önerilmiş
- Görev istenmediği kısa konuşma yanıtları
- Kullanıcının seçmesi için alternatifler sunulmuş

Yanıt JSON: `{"complete": true/false, "reason": "..."}`

**LLM başarısız olursa** (timeout, parse hatası, credentials yok) → regex fallback'e düşer.

### 5. Regex Fallback (`regex_fallback`) — Yedek Yol
LLM çağrısı başarısız olursa devreye girer.
Türkçe karakterleri normalize eder (ç→c, ğ→g, vb.) ve mesajın son 300 karakterinde yüksek-güven regex pattern'leri arar:

```
"hangisinden/nereden başlayalım"
"kodlamaya başlayalım mı"
"devam edeyim mi"
```

Sadece çok net olan durumları yakalar, false-positive'den kaçınır.

### 6. Loop Guard (`loop_guard`) — Sonsuz Döngü Koruması
Aynı mesajı iki kez bloklamamak için MD5 hash tutar.
Son 20 hash `~/.claude/hooks/.seen_blocks.json` dosyasında saklanır.
Eğer hash daha önce görüldüyse `exit 0` ile geçer — tekrar bloklama yapmaz.

### 7. Karar
**Tamamlanmamış tespit edildi:**
```python
sys.stderr.write(block_reason)
sys.exit(2)
```
`exit 0` + stdout JSON → Claude Code'a "dur, devam et" sinyali.
`{"decision":"block","reason":"..."}` → Claude'a feedback olarak iletilir.

**Tamamlanmış:**
```python
sys.exit(0)
```
Normal çıkış, Claude durabilir.

---

## Dosya Yapısı

| Dosya | Amaç |
|-------|-------|
| `~/.claude/hooks/check-completion.py` | Hook scripti |
| `~/.claude/hooks/check-completion.log` | Debug logları |
| `~/.claude/hooks/.seen_blocks.json` | Loop guard hash listesi |

---

## Stop Hook Protokolü (Claude Code)

Claude Code Stop hook'ları için exit kodları:
- `exit 0` → OK, Claude durabilir
- `exit 0` + stdout `{"decision":"block","reason":"..."}` → BLOCK, reason içeriği Claude'a feedback olarak iletilir ve Claude devam eder

Bu hook tam olarak bu protokolü kullanarak Claude'u "yarım bırakma, devam et" moduna sokar.

---

## Neden Bu Hook Var?

Claude bazen büyük bir görev alıp analiz yapar, plan yapar ama sonunda "Başlayayım mı?" diye sorar. Bu hook tam bu davranışı yakalar ve Claude'u izinsiz devam etmeye zorlar. Worker agent sisteminde bütün görevlerin tamamlanması kritik olduğu için bu hook merkezi bir role sahip.

---

## Özet Akış Diyagramı

```
Stop Hook tetiklenir
        ↓
Mesajı çıkar (last_assistant_message veya transcript)
        ↓
len < 30? → stdout "{}" + exit 0 (geç)
        ↓
LLM Check (Haiku via Rusk)
    ↓           ↓
 Başarılı    Başarısız
    ↓           ↓
complete?   Regex Fallback
    ↓           ↓
  TRUE      incomplete?
    ↓           ↓
stdout {}   Loop Guard
exit 0      already seen?
            ↓         ↓
        stdout {}   stdout {"decision":"block","reason":"..."}
          exit 0         exit 0
```

---

## Önemli: Claude Code Stop Hook API (2025+)

Eski API (`exit 2 + stderr`) artık çalışmıyor. Yeni API:

| Durum | stdout | exit code |
|-------|--------|-----------|
| Approve (geç) | `{}` | 0 |
| Block (devam et) | `{"decision": "block", "reason": "..."}` | 0 |

`exit 2` → "JSON validation failed" hatası verir. Her durumda `exit 0` kullan, stdout'a JSON yaz.

---

## /goal Komutu ile Etkileşim

`/goal` komutu Claude Code'un builtin özelliği — session-scoped stop hook ekler.

**⚠️ KULLANMA:** `/goal` komutu "JSON validation failed" hatasına yol açıyor. Condition text'i shell command olarak çalıştırmaya çalışıyor, bu başarısız olunca geçersiz JSON üretiyor.

**Kural:** Bu projede `/goal` kullanılmaz. `check-completion.py` zaten aynı işlevi (ve daha fazlasını) yapıyor.

Eğer yanlışlıkla `/goal` kullandıysan: `/goal clear` ile session hook'unu temizle, ardından yeni bir oturum aç.

**`check-completion.py` ne fark yapar:**
- `/goal`'den farklı olarak LLM semantic check + regex fallback + loop guard kullanır
- "Yarım bırakma" detection'ı daha güvenilir ve false-positive'den kaçınır
- Stop hook API'sine (stdout JSON + exit 0) tam uyumludur