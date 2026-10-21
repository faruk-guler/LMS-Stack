# Bölüm 4: Scalelite ile Yatay Ölçeklendirme ve Yük Dengeleme

BigBlueButton (BBB) doğası gereği "Monolithic" bir yapıya sahiptir; yani tek bir oda (meeting) tek bir sunucunun kaynaklarını kullanabilir. Ancak binlerce eşzamanlı kullanıcıya hizmet vermek için birden fazla BBB sunucusunu bir havuzda toplamak ve yükü dağıtmak gerekir. Bu noktada **Scalelite** devreye girer.

## 1. Scalelite Nedir?

Scalelite, BigBlueButton cluster (küme) yönetimi için açık kaynaklı bir yük dengeleyicidir. Birden fazla "Worker" (İşçi) BBB sunucusunu tek bir API uç noktası arkasında birleştirir.

* **Akıllı Yönlendirme:** Yeni bir toplantı isteği geldiğinde, en az yükü olan (CPU, kullanıcı sayısı veya toplantı sayısına göre) sunucuyu seçer.
* **Şeffaflık:** Önyüz (Moodle, Greenlight) sadece Scalelite ile konuşur. Arkadaki sunuculardan biri kapansa bile sistem çalışmaya devam eder.
* **Kayıt Yönetimi:** BBB sunucularında tamamlanan kayıtları merkezi bir depolama alanına (NFS/S3) aktarır ve oradan sunar.

## 2. Topoloji ve Sistem Gereksinimleri

Kurulum yapılacak olan örnek topolojimiz:

| Sunucu Rolü | Hostname / IP | İşletim Sistemi |
| :--- | :--- | :--- |
| **Moodle LMS** | myMoodle (20.44.12.226) | Ubuntu 24.04 |
| **Scalelite Load Balancer** | myscalelite (20.44.12.227) | Ubuntu 24.04 |
| **NFS Ortak Depolama** | mynfs (20.44.12.228) | Belirtilmedi |
| **BBB Room 01** | guler-room01 (20.44.12.229) | Ubuntu 24.04 (BBB 4.0) |
| **BBB Room 02** | guler-room02 (20.44.12.230) | Ubuntu 24.04 (BBB 4.0) |

### Minimum Gereksinimler

* **Docker:** En güncel sürümü.
* **Sistem:** 16 GB Bellek (Swap aktif), yüksek tek çekirdek performansına sahip 8 vCPU.
* **Disk:** Kayıtlar için en az 500 GB boş alan (NFS kullanılmıyorsa) veya 50 GB.
* **Ağ & Portlar:** Tümü için 250 Mbit/s simetrik bant genişliği.
  * **TCP:** 80 ve 443 (Reverse Proxy için boş olmalı).
  * **UDP:** 16384 - 32768.
* **DNS:** Sertifika kurulumu için geçerli bir hostname (örn: `myscalelite.guler.com`).
* **IP:** IPv4 ve IPv6 desteği.

## 3. Mimari Bileşenler

Scalelite kurulumu genellikle Docker üzerinden yapılır ve şu bileşenlerden oluşur:

| Bileşen | Görevi |
| :--- | :--- |
| **Scalelite API** | Uygulama mantığı, yük dengeleme hesaplamaları. |
| **Scalelite Poller** | BBB sunucularının sağlık durumunu ve yükünü periyodik kontrol eder. |
| **PostgreSQL** | Toplantı metadata ve durum bilgilerini saklar. |
| **Redis** | Pub/Sub ve caching işlemleri için kullanılır. |
| **Nginx** | Docker içi trafiği yönetir, SSL host Nginx üzerinden bağlanır. |

---

## 4. Scalelite Kurulum Adımları (Docker Compose)

### 4.1. Sunucu Hazırlığı ve Kurulum

Ubuntu 24.04 (myscalelite) üzerinde Docker ve gerekli sistem ön gereksinimlerini kuruyoruz:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose git openssl nginx
sudo systemctl enable --now docker
```

Scalelite yapılandırmasını daha düzenli tutmak için `/opt` dizinini kullanmak **best practice**'dir.

```bash
sudo mkdir -p /opt/scalelite
sudo chown $USER:$USER /opt/scalelite
cd /opt/scalelite
```

### 4.2. Güvenlik Anahtarlarının Üretilmesi

Scalelite çalışması için güçlü şifreleme anahtarlarına ihtiyaç duyar. Aşağıdaki `openssl` komutlarıyla bu değerleri üretin ve not edin:

* **SECRET_KEY_BASE** (64 Karakter):

    ```bash
    openssl rand -hex 64
    ```

* **LOADBALANCER_SECRET** (24 Karakter - Moodle/LMS ile paylaşacağımız secret):

    ```bash
    openssl rand -hex 24
    ```

* **Veritabanı Şifresi** (12 Karakter):

    ```bash
    openssl rand -hex 12
    ```

### 4.3. Çevresel Değişkenler (`.env`)

`/opt/scalelite/.env` dosyasını oluşturun ve ürettiğiniz değerlerle doldurun:

```bash
sudo nano /opt/scalelite/.env
```

```env
# --- GENEL AYARLAR ---
URL_HOST=myscalelite.guler.com
SECRET_KEY_BASE=üretilen_64_karakterlik_değer
LOADBALANCER_SECRET=üretilen_24_karakterlik_değer

# --- VERİTABANI VE REDIS BAĞLANTILARI ---
DB_PASSWORD=üretilen_veritabanı_şifresi

# PostgreSQL bağlantı URL'i (kullanıcı:şifre@host:port/veritabanı_adı)
DATABASE_URL=postgres://postgres:${DB_PASSWORD}@postgres:5432/scalelite_production
REDIS_URL=redis://redis:6379/0

# --- KAYITLAR ---
SCALELITE_RECORDING_DIR=/var/bigbluebutton/spool
```

> [!NOTE]
> `DATABASE_URL` içindeki şifre kısmına, `DB_PASSWORD` değerinizi `postgres:şifre_buraya@` şeklinde açık olarak veya `.env` üzerinden okuyacak formatta yazabilirsiniz.

### 4.4. `docker-compose.yml` Yapılandırması

Scalelite bileşenlerini ayağa kaldıracak olan dosyamızı yapılandırıyoruz. İçerisinde `redis`, `postgres`, `scalelite-api`, `poller`, `recording-importer` ve yükü karşılayacak dâhili `nginx` konteynerini kullanacağız.

```bash
sudo nano /opt/scalelite/docker-compose.yml
```

```yaml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis_data:/data

  postgres:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: scalelite_production
    volumes:
      - postgres_data:/var/lib/postgresql/data

  scalelite-api:
    image: ghcr.io/blindsidenetworks/scalelite-api:latest
    restart: always
    env_file: .env
    depends_on:
      - redis
      - postgres

  poller:
    image: ghcr.io/blindsidenetworks/scalelite-poller:latest
    restart: always
    env_file: .env
    depends_on:
      - redis
      - postgres

  recording-importer:
    image: ghcr.io/blindsidenetworks/scalelite-recording-importer:latest
    restart: always
    env_file: .env
    volumes:
      - /var/bigbluebutton/spool:/var/bigbluebutton/spool
    depends_on:
      - redis
      - postgres

  # Trafiği yönetecek dahili Nginx konteynerini
  nginx:
    image: ghcr.io/blindsidenetworks/scalelite-nginx:latest
    restart: always
    env_file: .env
    ports:
      - "3000:80"
    depends_on:
      - scalelite-api
    volumes:
      - /var/bigbluebutton/spool:/var/bigbluebutton/spool

volumes:
  redis_data:
  postgres_data:
```

Yapılandırmayı başlatalım:

```bash
cd /opt/scalelite
sudo docker-compose up -d
```

---

## 5. Host Nginx ve SSL Yapılandırması (Reverse Proxy)

Docker içerisindeki Nginx `3000` portundan hizmet vermektedir. Dış dünyadan (internetten) gelecek olan HTTPS (443) isteklerini karşılayıp güvenliğini sağlamak için host (ana) makinede bir Reverse Proxy ayarlamalıyız.

### 5.1. Sertifika Sağlama

`myscalelite.guler.com` için sertifikanın **üretilmesi (Let's Encrypt / ticari-küresel CA — Sectigo vb. / self-signed test), `fullchain` + ara (intermediate) zincirinin kurulması, `/etc/nginx/ssl/` altına yerleştirilmesi ve `dhparam` üretimi** dahil tüm adımların **tek kaynağı**: **Certs — Sertifika Kurulum Rehberi (`Certs/Sertifika_Rehberi.md`)**.

```bash
# Rehberdeki adımlar uygulandıktan sonra Scalelite host'unda cert'ler burada durur:
sudo mkdir -p /etc/nginx/ssl/
sudo openssl dhparam -out /etc/nginx/ssl/dhp-2048.pem 2048   # isteğe bağlı
# fullchain (sunucu + ara): /etc/nginx/ssl/myscalelite.guler.com.crt
# private key (chmod 600):   /etc/nginx/ssl/myscalelite.guler.com.key
```

### 5.2. Nginx Site Ayarı (`sites-available`)

Yönlendirme (Proxy) dosyasını oluşturuyoruz:

```bash
sudo nano /etc/nginx/sites-available/scalelite
```

```nginx
# 1. Kısım: HTTP (80) ile gelenleri zorla HTTPS'e yönlendir
server {
    listen 80;
    server_name myscalelite.guler.com;

    return 301 https://$server_name$request_uri;
}

# 2. Kısım: Güvenli HTTPS (443) kapısı ve Docker'a yönlendirme
server {
    listen 443 ssl;
    server_name myscalelite.guler.com;

    ssl_certificate /etc/nginx/ssl/myscalelite.guler.com.crt;   # TAM ZİNCİR (fullchain: sunucu + ara sertifikalar)
    ssl_certificate_key /etc/nginx/ssl/myscalelite.guler.com.key;
    ssl_dhparam /etc/nginx/ssl/dhp-2048.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    client_max_body_size 50M;

    # Gelen trafiği Docker'ın (3000) portuna aktarma
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Son adımda ayarları aktifleştirip servisi yeniden başlatıyoruz:

```bash
sudo ln -s /etc/nginx/sites-available/scalelite /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 6. Worker BBB Sunucularının Scalelite'a Eklenmesi

Scalelite yönetim merkezimizi kurundan sonra, BBB Worker düğümlerimizi kümeye ekleyeceğiz.

### 6.1. BBB Sunucusundan API Bilgilerini Almak

Eklemek istediğiniz BBB sunucusuna (Örn: `guler-room01`) SSH ile bağlanıp gizli anahtarı öğrenin:

```bash
# guler-room01 üzerinde çalıştırılacak:
sudo bbb-conf --secret
```

Çıktıdakilere benzer şu formattaki değerleri not alın:

* URL: `https://guler-room01.guler.com/bigbluebutton/`
* Secret: `Kp2GpQlrtrtrJusGKzxiHRo8`

### 6.2. Scalelite API'ye BBB Sunucusunu Ekleme

Tekrar Scalelite sunucunuza dönerek şu komutu çalıştırıp sunucuyu ekleyin:

```bash
# guler-room01 için:
docker compose exec scalelite-api bundle exec rake \
  "servers:add[https://guler-room01.guler.com/bigbluebutton/api,Kp2GpQlrtrtrJusGKzxiHRo8]"

# guler-room02 için:
docker compose exec scalelite-api bundle exec rake \
  "servers:add[https://guler-room02.guler.com/bigbluebutton/api,Vgtrtwgrwfdg42667t3wedfg]"
```

> [!WARNING]
> Sunucu eklendikten sonra doğrudan hizmet vermeye başlamaz! Eklenen sunucuların durumunu (Durumları: `offline` veya `disabled` olacaktır) öğrenmeniz gerekir.

### 6.3. Sunucu Durumunu Kontrol Etme ve Aktifleştirme

Sunucu ID'lerini ve durumlarını (status) görmek için:

```bash
docker compose exec scalelite-api bundle exec rake status
docker compose exec scalelite-api bundle exec rake servers
```

Ekranda (`rake status` / `rake servers`) çıkan sunucu **UUID**'sini kopyalayın ve trafiğe açın (`<ID>` kısmını gerçek UUID ile değiştirin, yoksa komut hata verir):

```bash
SERVER_ID=1a2b3c4d-5678-90ab-cdef-0123456789ab   # `rake servers` çıktısından kopyalayın
docker compose exec scalelite-api bundle exec rake "servers:enable[${SERVER_ID}]"
```

İsterseniz bir sunucuyu kaldırabilir veya yanlış bir girişi silebilirsiniz:

```bash
docker compose exec scalelite-api bundle exec rake "servers:remove[${SERVER_ID}]"
```

---

## 7. Moodle Yapılandırması ve Entegrasyon

Bütün yapı kurulduktan sonra Moodle sisteminizin artık tekil BBB sunucuları ile değil sadece Scalelite ile konuşması sağlanmalıdır:

1. Moodle admin paneline gidin `Yönetim -> Eklentiler -> Etkinlik modülleri -> BigBlueButton`.
2. **Sunucu URL:** `https://myscalelite.guler.com/bigbluebutton/api`
3. **Paylaşılan Parola:** Scalelite sunucusundaki `.env` dosyanızda yer alan `LOADBALANCER_SECRET` değeri.
4. **Sağlama Toplamı (Checksum) Algoritması:** BBB 3.x/4.0 **SHA256** kullanır; Moodle tarafında da varsayılan olarak **SHA256** bırakın. Yalnızca çok eski (2.x) bir worker'a bağlanıyorsanız geçici olarak SHA1 test edilebilir; aksi halde API çağrıları `checksum error` ile reddedilir.

> [!IMPORTANT]
> **Scalelite ve Büyük Odalar:** Scalelite "Tek bir odayı 10 sunucuya bölmez". Sadece odaları farklı sunuculara dağıtır. 500+ kişilik tek bir yayın için hala harici RTMP/Streaming çözümleri entegrasyonu önerilir.

---

## 8. Scalelite'ın Kendi Yedekliliği (HA)

Buraya kadar kurulan **tek Scalelite**, havuzun geri kalanı sağlıklı olsa bile **tek hata noktasıdır (SPOF)**: Scalelite düşerse Moodle/Greenlight API çağrılarının gidecek yeri kalmaz.

- Scalelite uygulaması **durumsuzdur (stateless)** — tüm durum dış **PostgreSQL** ve **Redis**'tedir. Bu yüzden aynı DB+Redis'e bakan **birden çok Scalelite** instance'ı çalıştırılabilir.
- Kurumsal desen: **2 Scalelite + keepalived VIP** (active/passive veya active/active); Scalelite'ın dış **PostgreSQL**'i **Patroni** ile, **Redis**'i **Sentinel** ile yedeklenir.

Tam topoloji, SPOF matrisi, config örnekleri ve failover runbook'ları için: **05 — Yüksek Erişilebilirlik (HA) & Failover (`05_High_Availability_and_Failover.md`)**.
