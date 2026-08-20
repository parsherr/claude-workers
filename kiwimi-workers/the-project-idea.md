> **Bağlantılar:** [[CLAUDE]] | [[agent-generator/core-idea]] | [[work-management-system-session-notes]]

# Kiwimi Workers — Proje Fikri

## Şirket Bağlamı
Kiwimi benim yazılım şirketim. Bu proje, Kiwimi'nin agent altyapısının temelini oluşturacak.

## Ana Fikir: Agent Kartları
Futbol kartları veya strateji oyunlarındaki general profilleri gibi düşün.

Her agent'ın bir **kimlik kartı** var:

| Alan | Açıklama |
|------|----------|
| İsim | Agent'ın adı |
| Profil resmi | Görsel kimlik |
| Açıklama | Ne iş yapar — tek cümle |
| Uzmanlık alanları | İyi olduğu konular listesi |
| Kullanım | Nasıl devreye alınır |

## Nasıl Kullanılır
1. İhtiyacın olan agent'ı katalogdan bul
2. `.md` dosyasını al
3. İstediğin projede, istediğin yerde kullan

## Bağlantı: Agent Generator
Agent'ları elle yazmak yerine [[agent-generator/core-idea]] sistemi devreye girer:

- Yeni bir agent'a ihtiyaç duyulduğunda Agent Generator'a söylenir
- Generator, kural ve pattern'lara göre optimal agent kartını oluşturur
- Kart doğrudan Kiwimi Workers kataloğuna eklenir

## Vizyon
Bir agent kütüphanesi: hazır, test edilmiş, birbirleriyle uyumlu çalışan
agent'lardan oluşan bir katalog. Projeye göre agent seçilir, konfigüre edilir, çalıştırılır.