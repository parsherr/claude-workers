# Work Management System — Session Notes

> **Bağlantılar:** [[CLAUDE]] | [[hooks]] | [[kiwimi-workers/the-project-idea]] | [[agent-generator/core-idea]]

**Tarih:** 2026-08-20
**Repo:** `parsherr/work-management-system`
**Lokasyon:** `/home/dogukan/Documents/github/work-management-system`

---

## Hedefimiz Ne?

Biz bu projede "bugün çalışan kod" değil, **10 yıl sonra da ayakta duran yazılım** inşa etmek istiyoruz.

Temel felsefe:
- Kod okunabilir ve anlaşılır olmalı — bir yabancı 10 saniyede ne yaptığını anlamalı
- Hızlı çözüm değil, doğru çözüm — hack'ler birikerek kodu öldürür
- Her fonksiyon tek bir iş yapmalı — "ve" kelimesi gerekiyorsa, bölünmesi gerekir
- Test olmadan bir özellik tamamlanmış sayılmaz — "it works" yetmez, passing test gerekir
- Yorum satırları "ne yaptığını" değil "neden öyle yaptığını" anlatmalı (English)
- Silinen kod gerçekten silinmeli — aynı commit'te, yoruma alınmış halde değil
- Dosyalar ~300 satırda tutulmalı, büyüyünce sorumluluğa göre bölünmeli
- Paylaşılan bir şeyi değiştirmeden önce kaç dosyanın import ettiğine bakılmalı

---

## Bu Conversationda Ne Yaptık?

### 1. CLAUDE.md Güncellendi

**Dosya:** `/home/dogukan/Documents/github/work-management-system/CLAUDE.md`

#### Eklenen bölümler:

**`## Non-Negotiable Session Rules`** — her oturumda Claude'un istisnasız uyması zorunlu 4 kural:
1. Maintainable & readable code
2. No dead code
3. Comments in English — always (Python docstring + TypeScript JSDoc zorunlu)
4. Tests are mandatory — test yoksa task tamamlanmış sayılmaz

**`## Engineering Philosophy`** — uzun vadeli mühendislik felsefesi (tam liste):
- Maintainability over quick fixes
- File size discipline (~300 satır; `files.py` → `files_*.py` split referans pattern)
- Readable, not clever
- Documentation & comments (why not what, English, docstring/JSDoc format)
- Explicit interfaces (Python full type annotation, TypeScript no `any`)
- Test before done ("it works is not verification, a passing test is")
- No dead code (aynı commit'te silinmeli)
- Change blast radius (paylaşılan utility/type değiştirilmeden önce import sayısına bakılır)

**File router split** güncellendi — `files.py` monolitiği bölündü:
- `files_core.py` — list, upload, download, rename, move, copy
- `files_bulk.py` — bulk move, copy, trash
- `files_trash.py` — trash management
- `files_share.py` — share links
- `files_drive.py` — Google Drive import (SSE)
- `files_misc.py` — quota, zip, search, star, recent
- `files_utils.py` — shared helpers

**`There are no tests in this project.`** satırı silindi — artık test yazıyoruz.

#### CLAUDE.md'deki mevcut kritik kurallar (değiştirilmedi ama önemli):
- **NEVER start dev servers** — `pnpm dev`, `uvicorn`, `npm start` asla çalıştırılmaz; user yönetir
- **NEVER run `pnpm typecheck`** — makineyi çökertir; sadece kullanıcı talep ederse çalıştırılır
- `pnpm lint` çalıştırılabilir; uzun shell pipeline'larına sokulmaz

---

### 2. Agent Dosyaları Yeniden Yazıldı

**Klasör:** `/home/dogukan/Documents/github/work-management-system/.claude/agents/`

#### Problem: Tüm agent'lar yanlış projeyi (Postgrify) referans ediyordu
- Postgrify: Fastify + TypeScript, `packages/api/`, `packages/gui/`, TypeScript
- Bu proje: FastAPI + Python backend, Next.js App Router frontend

#### Yapılan:

| Agent | Durum | Ne Yapıldı |
|-------|-------|------------|
| `lead.md` | ✅ Yeniden yazıldı | Bu projeye adapte orchestrator |
| `backend.md` | ✅ Yeniden yazıldı | FastAPI + SQLModel + Python |
| `frontend.md` | ✅ Yeni oluşturuldu | Next.js App Router (daha önce yoktu!) |
| `security.md` | ✅ Yeniden yazıldı | Bu projenin 4 katman threat modeli |
| `tester.md` | ✅ Yeniden yazıldı | pytest + bu projenin endpoint'leri |
| `releaser.md` | ✅ Yeniden yazıldı | `parsherr/work-management-system` repo |
| `release.md` | ❌ Silindi | `releaser.md` ile duplikattı |
| `frontend-designer.md` | ✓ Dokunulmadı | Sadece görsel, logic'e dokunmaz, uygundu |

---

## Lead Agent — Tam Çalışma Mantığı

### Temel Kural: Kullanıcıyla Sadece 2 Kez Konuşur
1. Tüm iş tamamen bitince → özet rapor sunar ve durur
2. Bir şey fiziksel olarak imkansızsa → kısa açıklar ve durur

**Asla sormaz:**
- "Devam edeyim mi?"
- "Bu yaklaşım uygun mu?"
- "Onaylıyor musunuz?"

Görevi alır → planı sessizce yapar → çalıştırır → raporlar.

### Agent'ların Birbirleriyle Konuşması (SendMessage)

Lead agent, alt agent'larla `SendMessage` tool'u üzerinden iletişim kurar:

```
Lead → SendMessage("backend", görev_promptu)
Backend çalışır → rapor döner
Lead raporu okur → kalite barını kontrol eder
  ├── GEÇTI → SendMessage("security", context_ile_güvenlik_görevi)
  └── GEÇMEDİ → yeni prompt yazar → SendMessage("backend", yeni_prompt)
```

Her agent kendi uzmanlık alanında çalışır, birbirinin alanına girmez:
- `backend` → Python/FastAPI dosyaları
- `frontend` → Next.js/TypeScript dosyaları
- `security` → kod incelemesi ve hardening
- `tester` → test dosyaları yazmak ve çalıştırmak
- `releaser` → git tag, CHANGELOG, GitHub Release

### Kalite Barı — Lead Bir Raporu Ne Zaman Reddeder?

**Backend raporunu reddet:**
- Test dosyası yoksa (`backend/tests/test_<module>.py`)
- Type annotation eksikse herhangi bir fonksiyon imzasında
- Fonksiyon birden fazla iş yapıyorsa ("ve" ile açıklanıyorsa)
- Edge case'ler handle edilmemişse (401, 403, 404, 422, DB error)
- Docstring eksikse non-trivial fonksiyonlarda
- Dead code veya comment'e alınmış bloklar varsa
- Model değişikliği için Alembic migration oluşturulmamışsa

**Frontend raporunu reddet:**
- `any` varsa TypeScript'te
- Sidebar layout wrapper eksikse yeni sayfada
- Global context hook'ları bypass edilerek prop-drilling yapılmışsa
- Server Action'da error handling yoksa
- JSDoc eksikse props interface'inde
- Dead import veya unused variable varsa

**Security raporunu reddet:**
- "No issues found" diyorsa kanıt göstermeden
- 4 katmanın hepsi incelenmemişse (network/input, auth/token, RBAC, filesystem/R2)
- Severity classification yoksa (Critical/High/Medium/Low)
- Her bulgu için: vulnerable snippet + attack scenario + fix + test yoksa

**Tester raporunu reddet:**
- Sadece happy path varsa (401, 403, 404, 422 senaryoları eksikse)
- Test cleanup yapılmamışsa
- Sayısal sonuç yoksa ("7/7 pass" gibi — "çalıştı" kabul edilmez)

### Re-Prompt Format — Lead Yeniden Prompt Nasıl Yazar?

Lead bir raporu reddettiğinde yeni prompt şu formatta olmalı:

```
## Previous Work — Gaps

[Somut eksikler — "yetersiz" değil, tam olarak ne eksik:]
- test_files_share.py dosyası oluşturulmamış
- create_share() fonksiyonunda docstring yok
- DB error senaryosu (session.exec raises) handle edilmemiş

## What To Do This Time

[Her eksikliği kapatacak spesifik talimatlar]

## Context

[Önceki rapordan çalışan kısımlar — bunları tekrar yapma]

## Expected Output

[Hangi dosyalar, hangi testler, hangi kontroller]
```

### Execution Flows

**Standard Feature Flow (backend + frontend birlikte):**
```
1. backend agent → implement
2. security agent → review (4 katman)
3. [security bulgusu varsa] backend agent → fix
4. frontend agent → UI/integration
5. tester agent → backend + frontend tests
6. Kullanıcıya final rapor → dur
```

**Backend-Only Task:**
```
1. backend → implement
2. security → review
3. [fix gerekirse] backend → fix
4. tester → test
5. Rapor → dur
```

**Frontend-Only Task:**
```
1. frontend → implement
2. security → review (auth gating, RBAC, XSS)
3. tester → integration test
4. Rapor → dur
```

**Security Audit:**
```
1. security → full audit (4 katman)
2. [Critical/High] backend → fix
3. [Critical/High] frontend → fix
4. tester → regression test
5. Rapor → dur
```

**Release:**
```
1. releaser → bump version, CHANGELOG, tag, push, GitHub Release
2. Rapor → dur
```

### Final Rapor Formatı (Lead'den kullanıcıya)

```
## ✅ Done: [Task Name]

### What Was Done
- backend: [özet, değişen dosyalar]
- frontend: [özet, değişen dosyalar]
- security: [N bulgu, N kapatıldı]
- tester: [N test, N pass / N fail]

### Evaluation Cycles
- backend: attempt N'de geçti
- security: attempt N'de geçti

### Key Decisions
- [mimari kararlar, trade-off'lar]

### Failed Steps  ← sadece bir şey başarısız olduysa
```

---

## Test Yazım Standartları

### Backend (pytest)

**Konum:** `backend/tests/test_<module>.py`

Örnekler: `test_files_core.py`, `test_auth.py`, `test_tasks.py`

**Her endpoint için zorunlu senaryolar:**
1. Happy path (200 + doğru response shape)
2. Auth failure → 401
3. RBAC failure → 403
4. Not found → 404
5. Invalid input → 422
6. Edge case (boş liste, duplicate, boundary value)

**Fixture pattern:**
```python
@pytest.fixture(name="session")
def session_fixture():
    """In-memory SQLite session for isolated test runs."""
    engine = create_engine("sqlite:///:memory:")
    SQLModel.metadata.create_all(engine)
    with Session(engine) as session:
        yield session

@pytest.fixture
def uploaded_file(client, auth_headers):
    """Create test file; clean it up after test — always."""
    response = client.post("/api/v1/files/upload", ...)
    file_id = response.json()["id"]
    yield file_id
    client.delete(f"/api/v1/files/trash/{file_id}", headers=auth_headers)
```

**Çalıştırma:**
```bash
cd backend && source .venv/bin/activate && pytest
cd backend && source .venv/bin/activate && pytest tests/test_files_core.py -v
```

### Frontend
**Konum:** `frontend/tests/<component>.test.tsx`

---

## Security Agent — 4 Katman Threat Modeli

Bu projeye özgü güvenlik kontrol katmanları:

1. **Network & Input Boundary** — unauthenticated endpoint'ler, file upload (2GB limit), Google Drive import, query param injection
2. **Auth & Token Lifecycle** — JWT access+refresh, `proxy.ts` middleware, `has_session`/`is_admin`/`user_role` cookie'leri, `NEXT_PUBLIC_MOCK_AUTH=true` asla production'da olmamalı, localStorage JWT (XSS riski — mimari sorun olarak işaretle)
3. **Authorization & RBAC** — admin kontrolü BOTH `is_admin` AND `role=="admin"` olmalı, frontend RBAC display-only (backend bağımsız enforce etmeli), `/admin` route `proxy.ts`'de gated
4. **Filesystem & Storage** — `getSafePath()` her file operation'da çağrılmalı, R2 key'ler `{user_id}/{path}` ile scoped, Google Drive import user namespace'inde, share token'ları `secrets.token_urlsafe`

---

## Öğrenilen Dersler

1. **Agent dosyaları projeye özgü olmalı.** Başka projeden (Postgrify) kopyalanmış agent'lar yanlış stack, yanlış path, yanlış kalite barı getirir. Her projede sıfırdan yazılmalı.

2. **CLAUDE.md'deki kurallar `## Non-Negotiable Session Rules` olarak başa yazılmazsa Claude her oturumda uygulamaz.** Bu bölüm bunu zorluyor.

3. **"There are no tests in this project" gibi tek bir satır bile Claude'un davranışını etkiler.** Sildik — artık test yazıyoruz.

4. **Frontend logic agent'ı yoksa lead'in frontend işi yapacak biri yoktu.** `frontend-designer.md` sadece görsel. `frontend.md` eklendi.

5. **Duplicate agent'lar (`release.md` + `releaser.md`) lead'i karıştırır.** Biri silindi.

6. **Kalite barı somut olmalı.** "Yetersiz" değil, "test dosyası yok", "docstring eksik", "any kullanılmış" gibi spesifik kriterler.

7. **Lead'in re-prompt formatı standart olmalı.** Somut gap listesi + ne yapılacak + context + expected output — bu olmadan agent aynı hatayı tekrarlar.

8. **Her flow'un adımları önceden tanımlanmış olmalı.** Lead "ne zaman security'e göndereyim" diye düşünmemeli — flow şeması bunu söyler.

---

## Referans Dosyalar

| Dosya | Açıklama |
|-------|---------|
| `/home/dogukan/Documents/github/work-management-system/CLAUDE.md` | Ana proje talimatları — her oturumda okunur |
| `.claude/agents/lead.md` | Orchestrator — görevi buraya ver |
| `.claude/agents/backend.md` | FastAPI + SQLModel backend engineer |
| `.claude/agents/frontend.md` | Next.js App Router frontend engineer |
| `.claude/agents/security.md` | 4 katman security reviewer |
| `.claude/agents/tester.md` | pytest QA, GitHub issue açar |
| `.claude/agents/releaser.md` | Semver + CHANGELOG + GitHub Release |
| `.claude/agents/frontend-designer.md` | Görsel UI designer (logic yazmaz) |

---

## Başka Projeye Bu Setup'ı Taşımak İçin Checklist

- [ ] `.claude/agents/lead.md` — projenin stack'ini, repo adını, test path'lerini güncelle
- [ ] `.claude/agents/backend.md` — backend framework, venv path, migration aracı
- [ ] `.claude/agents/frontend.md` — framework, routing pattern, state management, UI lib
- [ ] `.claude/agents/security.md` — projenin auth mekanizması, storage, threat surface
- [ ] `.claude/agents/tester.md` — test framework, endpoint listesi, repo adı
- [ ] `.claude/agents/releaser.md` — repo adı, version dosyaları, branch adı
- [ ] `CLAUDE.md` — `## Non-Negotiable Session Rules` bölümü mutlaka var olmalı
- [ ] `CLAUDE.md` — "There are no tests" gibi yanlış yönlendiren satırlar silinmeli
- [ ] Duplicate agent'lar temizlenmeli