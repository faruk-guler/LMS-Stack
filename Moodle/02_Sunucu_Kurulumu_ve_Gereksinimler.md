# 02 - Sistem Kurulumu ve Sunucu Gereksinimleri (Moodle 5.3 LTS)

## 2.1 Donanım Gereksinimleri

| Ölçek | CPU | RAM | Disk | Ağ |
| :--- | :--- | :--- | :--- | :--- |
| Küçük (< 500 Kullanıcı) | 4 Core | 8 GB | 100 GB NVMe | 100 Mbps |
| Orta (500–5000 Kullanıcı) | 8 Core | 16 GB | 500 GB NVMe | 1 Gbps |
| Büyük (5000+ Kullanıcı) | 16+ Core | 32+ GB | 1 TB NVMe + ext. storage | 1–10 Gbps |

> [!WARNING]
> Sınav haftalarında Moodle normal trafiğin 5-10 katı kaynak tüketebilir. Donanım seçimini ortalama değil, **pik** trafiğe göre yapın.

## 2.2 Yazılım Gereksinimleri (Moodle 5.3 — Ekim 2026)

| Bileşen | Minimum | Önerilen |
| :--- | :--- | :--- |
| İşletim Sistemi | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| PHP | **8.3** (64-bit zorunlu) | **8.3** veya **8.4** |
| PostgreSQL | **17** | **17+** |
| Redis | 6.x | **7.x** |
| Web Server | Apache / Nginx | **Nginx** |

> [!IMPORTANT]
> Moodle 5.3, yalnızca **64-bit PHP** destekler. `php -r "echo PHP_INT_SIZE;"` çıktısı `8` olmalıdır (64-bit sistemde bir tamsayı 8 **bayt** = 64 bit yer kaplar). `4` görüyorsanız 32-bit PHP kurulumu var demektir.

## 2.3 PHP ve Bağımlılıkların Kurulumu

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install -y nginx postgresql-17 redis-server \
  php8.3-fpm php8.3-pgsql php8.3-curl php8.3-gd php8.3-intl \
  php8.3-mbstring php8.3-xml php8.3-xmlrpc php8.3-zip \
  php8.3-soap php8.3-opcache php8.3-redis php8.3-sodium \
  php8.3-bcmath php8.3-imagick php8.3-tokenizer
```

> [!NOTE]
> `php8.3-sodium` Moodle 5.x'de **zorunlu** bir bağımlılıktır. Kurulmadan Moodle installer başlamaz. `max_input_vars` ayarının **5000 veya üstü** olması zorunludur.

## 2.4 PHP İnce Ayarları (php.ini)

```bash
sudo nano /etc/php/8.3/fpm/php.ini
```

```ini
memory_limit = 512M
upload_max_filesize = 128M
post_max_size = 128M
max_execution_time = 300
max_input_vars = 5000         ; Moodle 5.x için zorunlu minimum
cgi.fix_pathinfo = 0
date.timezone = Europe/Istanbul
```

## 2.5 PHP-FPM Havuz Ayarları

```bash
sudo nano /etc/php/8.3/fpm/pool.d/moodle.conf
```

```ini
[moodle]
user = www-data
group = www-data
listen = /run/php/php8.3-fpm-moodle.sock
listen.owner = www-data
listen.group = www-data

pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 3
pm.max_spare_servers = 10
pm.max_requests = 500

php_admin_value[error_log] = /var/log/php-fpm/moodle-error.log
```

## 2.6 Nginx Yapılandırması (Moodle 5.x — /public Dizini!)

> [!IMPORTANT]
> Moodle 5.0'dan itibaren Nginx `root` direktifi **`/public`** alt dizinine yönlendirilmelidir.

```bash
sudo nano /etc/nginx/sites-available/moodle
```

```nginx
server {
    listen 80;
    server_name moodle.sirket.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name moodle.sirket.com;

    # Moodle 5.x: root artık /public alt dizini!
    root /var/www/moodle/public;
    index index.php;

    ssl_certificate     /etc/letsencrypt/live/moodle.sirket.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/moodle.sirket.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # moodledata'ya dışarıdan erişimi engelle
    location /dataroot/ {
        internal;
        alias /var/www/moodle/moodledata/;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm-moodle.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_read_timeout 300;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

> [!NOTE]
> Kendi SSL sertifikanızı (Let's Encrypt, **ticari/küresel CA — Sectigo vb.**, veya test amaçlı self-signed) kullanacaksanız: **üretim, `fullchain` + ara (intermediate) zinciri, `/etc/nginx/ssl/`'e kurulum ve `chmod 600` anahtar izinleri dahil tüm adımların tek kaynağı Certs — Sertifika Kurulum Rehberi (`Certs/Sertifika_Rehberi.md`)** belgesidir. Moodle'ya özgü ek gereksinim: `config.php` içindeki `$CFG->wwwroot` mutlaka `https://moodle.sirket.com` olmalı ve `ssl_certificate` **tam zinciri (fullchain)** göstermeli (yoksa tarayıcı/WebSocket TLS uyarıları oluşur).

## 2.7 PostgreSQL Kurulumu

```bash
sudo -u postgres psql
```

```sql
CREATE USER moodleuser WITH PASSWORD 'cok_guclu_bir_sifre';
CREATE DATABASE moodledb
  WITH OWNER moodleuser
  ENCODING 'UTF8'
  LC_COLLATE 'tr_TR.UTF-8'
  LC_CTYPE 'tr_TR.UTF-8'
  TEMPLATE template0;
GRANT ALL PRIVILEGES ON DATABASE moodledb TO moodleuser;
\q
```

## 2.8 Moodle Kodlarının Kurulumu (Git Yöntemi)

```bash
# Moodle 5.3 (LTS) stabil dalını klonla
cd /var/www
sudo git clone -b MOODLE_503_STABLE https://github.com/moodle/moodle.git moodle

# moodledata dizini web root (/public) DIŞINDA olmalıdır
sudo mkdir -p /var/www/moodle/moodledata
sudo chown -R www-data:www-data /var/www/moodle/moodledata
sudo chmod -R 750 /var/www/moodle/moodledata

# Moodle kodu root:www-data sahipliğinde olmalı (Kök dizin)
sudo chown -R root:www-data /var/www/moodle
sudo chmod -R 755 /var/www/moodle
# public alt dizinine Nginx erişimi için izin ver
sudo chmod -R 755 /var/www/moodle/public
sudo chmod 640 /var/www/moodle/config.php

sudo ln -s /etc/nginx/sites-available/moodle /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Kurulumu tamamlamak için tarayıcıdan `https://moodle.sirket.com/install.php` adresine girin.

> [!NOTE]
> Moodle kurulum sihirbazını GUI üzerinden bitirdiğinizde **Varsayılan Admin Hesabı** tanımlaması istenir. Geleneksel yapıda aşağıdaki gibi standart oluşturulabilir:
>
> * **Kullanıcı Adı:** `admin`
> * **Şifre:** `Pass3556++!!` (veya sizin belirlediğiniz güçlü bir şifre)
> * **Site Adı:** `GULER Uzaktan Eğitim Sistemi`

### 2.9 Varsayılan Portlar ve Erişim

Kurumsal Firewall kuralları için aşağıdaki portların açık olduğundan emin olun:

* **HTTP/HTTPS:** 80, 443
* **PostgreSQL:** 5432
* **Redis:** 6379

---

> [!TIP]
> Moodle 5.3 (LTS) için Git branch adı `MOODLE_503_STABLE`'dır. 5.2 için `MOODLE_502_STABLE`, 4.5 LTS için `MOODLE_405_STABLE` şeklinde devam etmektedir.
