# Bölüm 15: BigBlueButton 4.0 — Yenilikler ve SysAdmin Notları

Ekim 2026 itibarıyla **BigBlueButton 4.0** kararlı (stable) sürümdür (v4.0.0, 14 Ekim 2026) ve bu rehberin varsayılan hedefidir. 2.x ve 3.x sürümleri terk edilmiş/eski hattır; yalnızca 3.x'te kalan kurumlar için migrasyon notları ilgili bölümlerde verilmiştir.

> [!IMPORTANT]
> **BBB 4.0 → Ubuntu 24.04 LTS + Docker** gerektirir ve medya tarafında **LiveKit varsayılandır**. 3.0'dan 4.0'a **yerinde (in-place) yükseltme desteklenmez**; temiz kurulum + kayıt migrasyonu gerekir.

## 15.1 Temel Mimari Değişimler (2.x → 3.x → 4.0)

Aşağıdaki tablo, 4.0'a gelene kadar olan evrimi özetler; **kalın sütun bu rehberin hedefi olan 4.0**'tır.

| Değişiklik | BBB 2.x | BBB 3.x | BBB 4.0 |
| :--- | :--- | :--- | :--- |
| Medya Sunucusu | Kurento + Mediasoup | **Sadece Mediasoup** | **LiveKit (varsayılan)**, Mediasoup/FreeSWITCH seçimlik |
| Beyaz Tahta | Eski bitmap tabanlı | **TLDraw** (vektörel) | **TLDraw** |
| Paylaşılan Notlar | Etherpad | Etherpad | **BlockNote** (Etherpad kaldırıldı) |
| Eklenti Sistemi | Yok | **Plugin Architecture** (3.0) | **Plugin SDK 1.0 / App Gallery** |
| Greenlight | v2 (Rails) | **v3 (React + PostgreSQL)** | **v3 (React + PostgreSQL)** |
| Taban (OS / Dil) | Ubuntu 20.04 | Ubuntu 22.04 · Java 11 | **Ubuntu 24.04 + Docker · Java 21 / Node.js 24 / Grails 8** |
| MongoDB Rolü | Gerekli | Hâlâ gerekli (aktif toplantı durumu) | Hâlâ gerekli |

**BBB 4.0'ın öne çıkan kullanıcı/arayüz yenilikleri:**

- Yeniden tasarlanan istemci: navigasyon kenar çubuğu (sidebar) ve **Apps Gallery** (uygulama/eklenti vitrini)
- Kullanıcı listesinde arama, yeniden tasarlanan zamanlayıcı (timer), adanmış misafir yönetim paneli ve lock-viewers modalü
- İşletim sistemini izleyen otomatik **açık/koyu tema**; gelişmiş ses filtreleme (WASM gürültü bastırma)
- **Request to Present:** izleyiciler moderatörlük talep edebilir
- Sabitlenen (pinned) sohbet mesajları, çok işlevli (Multi-Functional) yan panel modu
- **Breakout odaları için Learning Dashboard**; **BlockNote** tek ortak-not editörü (Etherpad kaldırıldı)
- Altta yatan taban: **Node.js 24, Java 21, Grails 8, Spring Boot 4** ve API tarafında **GraphQL**

## 15.2 Yeni TLDraw Beyaz Tahta

BBB 3.0 ile birlikte modern **TLDraw** altyapısına geçilmiştir:

- **Vektörel Çizim:** Nesneler seçilebilir, taşınabilir, gruplanabilir
- **Gelişmiş Araçlar:** Yapışkan notlar, akıllı oklar, pürüzsüz serbest çizim
- **Daha Az Bant Genişliği:** Olaylar NodeJS ve Redis üzerinden optimize edilmiş paketlerle iletilir

## 15.3 Plugin SDK 1.0 / App Gallery (BBB 4.0)

BBB 3.0'da tanıtılan ve **4.0'ta olgunlaşan** eklenti mimarisi: **sunucu tarafında değişiklik yapmadan** yeni özellikler eklenebilir. 4.0 ile **Plugin SDK 1.0** ve odada uygulama sabitleme/arama imkânı veren **App Gallery** gelir.

```bash
# Mevcut yüklü eklentileri görüntüle
ls /etc/bigbluebutton/bbb-html5-with-plugin-support/

# Resmi plugin deposu (SDK):
# https://github.com/bigbluebutton/bigbluebutton/tree/v4.0.x-release/bigbluebutton-html-plugin-sdk
```

## 15.4 Greenlight v3 Yenilikleri

```bash
# Greenlight Docker image sürümünü kontrol et
cd ~/greenlight-v3

# Yöntem 1: Docker image tag'inden sürümü oku
docker compose images greenlight

# Yöntem 2: Konteyner içinden Rails ortam bilgisini oku
docker compose exec greenlight bundle exec rake greenlight:version 2>/dev/null || \
  docker inspect greenlight-v3-greenlight-1 --format '{{ index .Config.Labels "org.opencontainers.image.version" }}'

# Greenlight'ı en güncel v3 imajına çekmek için
docker compose pull
docker compose up -d
```

Greenlight v3 ile gelen yenilikler:

- Basitleştirilmiş arayüz (daha az adımda oda oluşturma)
- Gelişmiş oda ayarları (katılımcı limiti, bekleme odası zorunluluğu)
- OIDC (OpenID Connect) desteği geliştirildi

## 15.5 Kamera ve Sanal Arka Plan

- **Sanal Arka Plan:** Kamera flulaştırma ve özel arka plan ekleme **istemci tarafında** (Client-Side) işlenir — sunucuya ekstra CPU yükü binmez
- **SysAdmin Notu:** Kurumsal arka plan görsellerini `/var/www/bigbluebutton-default/backgrounds/` dizinine ekleyerek kullanıcı panelinde otomatik sunabilirsiniz

## 15.6 Moderatör Araçları

- **Dikkat Takibi:** BBB sekmesinden uzaklaşan kullanıcılar moderatöre görünür
- **Toplu Yönetim:** Breakout odalarından ana odaya tek komutla geri çekme
- **Hard-Mute:** Tüm mikrofonları anında kapatma
- **API Tepki Süreleri:** Milisaniye seviyesine indirilmiştir

## 15.7 Ubuntu 24.04 Standartları (BBB 4.0)

```bash
# BBB 4.0 için sistem kontrolü
lsb_release -a              # Ubuntu 24.04 (Noble) olmalı
docker --version            # 4.0 LibreOffice dönüşümü için Docker zorunlu
bbb-conf --version          # 4.0.x
bbb-conf --status           # Tüm servisler active/running olmalı
```

> [!NOTE]
> **Migrasyon:** Hâlâ bir 3.0.x sunucusu çalıştırıyorsanız, 4.0'a geçiş için yeni bir **Ubuntu 24.04** sunucu kurup (`-v noble-400`, `v4.0.x-release` dalı) kayıtları taşıyın; eski sunucuyu yerinde yükseltmeye çalışmayın.

---

> [!TIP]
> **2026 Yol Haritası:** **BBB 4.0 (Ekim 2026)**; Ubuntu 24.04, Docker tabanlı LibreOffice dönüşümü, LiveKit medya varsayılanı, App Gallery/Plugin SDK ve BlockNote ile birlikte **yayınlandı**. Production'a alırken LiveKit medya seçimini, kayıt formatlarını ve API/eklenti (plugin) uyumluluğunu mutlaka test edin.
