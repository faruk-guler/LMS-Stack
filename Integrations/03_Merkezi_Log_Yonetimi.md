# Bölüm 3: Merkezi Log Yönetimi — Moodle & BigBlueButton (2026)

Moodle ve BigBlueButton birlikte çalıştığında onlarca farklı servisin logunu takip etmek gerekir. Her sunucuya SSH açıp `tail -f` yapmak ölçeklenebilir değildir. Bu bölüm, tüm logları merkezi bir noktada toplama, sorgulama ve alarm oluşturma yöntemlerini kapsar.

## 1. Log Kaynakları — Hızlı Başvuru Haritası

### Moodle Tarafı

| Servis | Log Konumu | İçerik |
| :--- | :--- | :--- |
| Nginx erişim | `/var/log/nginx/access.log` | HTTP istekleri, durum kodları |
| Nginx hata | `/var/log/nginx/error.log` | 502, 404, izin hataları |
| PHP-FPM | `/var/log/php-fpm/moodle-error.log` | PHP fatal, memory, timeout |
| PostgreSQL | `/var/log/postgresql/postgresql-17-main.log` | Slow query, deadlock |
| Redis | `/var/log/redis/redis-server.log` | Bağlantı, bellek uyarıları |
| Moodle Cron | `journalctl -u moodle-cron` | Cron çıktıları ve hataları |
| Moodle Uygulama | `Admin > Reports > Live logs` | Kullanıcı eylemleri (DB) |

### BigBlueButton Tarafı

| Servis | Log Konumu | İçerik |
| :--- | :--- | :--- |
| bbb-web (API) | `/var/log/bigbluebutton/bbb-web.log` | Toplantı API hataları |
| bbb-html5 | `/var/log/bbb-html5/bbb-html5.log` | Frontend/NodeJS hataları |
| bbb-webrtc-sfu | `/var/log/bbb-webrtc-sfu/bbb-webrtc-sfu.log` | Kamera/ekran paylaşımı (Mediasoup) |
| FreeSWITCH | `/opt/freeswitch/var/log/freeswitch/freeswitch.log` | Ses, SIP, 1007/1020 hataları |
| Nginx | `/var/log/nginx/error.log` | 502, WebSocket hataları |
| MongoDB | `/var/log/mongodb/mongod.log` | bbb-html5 veri deposu |
| Redis | `/var/log/redis/redis-server.log` | Pub/Sub mesaj kuyruğu |
| Kayıt İşleme | `/var/log/bigbluebutton/bbb-rap-worker.log` | Recording pipeline hataları |

### PILOS Tarafı (Opsiyonel Ön Yüz)

| Servis | Log Konumu | İçerik |
| :--- | :--- | :--- |
| PILOS App | `docker logs pilos_app` | Laravel / Uygulama hataları |
| PILOS DB | `docker logs pilos_db` | PostgreSQL / MariaDB logları |
| PILOS Nginx | `docker logs pilos_nginx` | HTTP istekleri ve hataları |
| Dosya Logları | `./storage/logs/*.log` | Uygulama bazlı günlük dosyalar |

---

## 2. Strateji 1: Loki + Grafana (Hafif ve Modern)

Loki, Grafana Labs'ın geliştirdiği **log toplama** aracıdır. Elasticsearch'e kıyasla çok daha az kaynak tüketir çünkü log içeriğini indekslemez, sadece etiketleri (labels) indeksler.

### 2.1 Mimari

```text
┌─────────────────┐     ┌─────────────────┐
│   Moodle Srv.   │     │   BBB Srv.      │
│  (Promtail)     │     │  (Promtail)     │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     ▼
           ┌─────────────────┐
           │      Loki       │
           │  (Log Storage)  │
           └────────┬────────┘
                    ▼
           ┌─────────────────┐
           │    Grafana       │
           │  (Görselleştirme │
           │   & Alarmlar)   │
           └─────────────────┘
```

### 2.2 Loki + Grafana Kurulumu (Monitoring Sunucusu)

```bash
# Docker Compose ile Loki + Grafana kurulumu
sudo mkdir -p /opt/monitoring && cd /opt/monitoring
sudo nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  loki:
    image: grafana/loki:3.4
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  grafana:
    image: grafana/grafana:11.6
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=GucluBirSifre123!
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - loki
    restart: unless-stopped

volumes:
  loki_data:
  grafana_data:
```

```bash
docker compose up -d
# Grafana: http://monitoring.sirket.com:3000 (admin / GucluBirSifre123!)
```

### 2.3 Promtail Kurulumu (Her Sunucuda)

Promtail, log dosyalarını okuyup Loki'ye gönderen hafif bir ajandır. Hem Moodle hem BBB sunucusuna kurulur.

```bash
# Promtail binary indir (Linux amd64)
PROMTAIL_VERSION=$(curl -s https://api.github.com/repos/grafana/loki/releases/latest | grep tag_name | cut -d '"' -f 4)
sudo wget -O /tmp/promtail.zip \
  "https://github.com/grafana/loki/releases/download/${PROMTAIL_VERSION}/promtail-linux-amd64.zip"
sudo unzip /tmp/promtail.zip -d /usr/local/bin/
sudo chmod +x /usr/local/bin/promtail-linux-amd64
sudo ln -sf /usr/local/bin/promtail-linux-amd64 /usr/local/bin/promtail
```

**Moodle Sunucusu için Promtail yapılandırması:**

```bash
sudo mkdir -p /etc/promtail
sudo nano /etc/promtail/config.yml
```

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://monitoring.sirket.com:3100/loki/api/v1/push

scrape_configs:
  # Nginx logları
  - job_name: moodle-nginx
    static_configs:
      - targets: [localhost]
        labels:
          job: moodle
          service: nginx
          __path__: /var/log/nginx/*.log

  # PHP-FPM logları
  - job_name: moodle-phpfpm
    static_configs:
      - targets: [localhost]
        labels:
          job: moodle
          service: php-fpm
          __path__: /var/log/php-fpm/moodle-error.log

  # PostgreSQL logları
  - job_name: moodle-postgresql
    static_configs:
      - targets: [localhost]
        labels:
          job: moodle
          service: postgresql
          __path__: /var/log/postgresql/*.log

  # Redis logları
  - job_name: moodle-redis
    static_configs:
      - targets: [localhost]
        labels:
          job: moodle
          service: redis
          __path__: /var/log/redis/*.log
```

**BigBlueButton Sunucusu için Promtail yapılandırması:**

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://monitoring.sirket.com:3100/loki/api/v1/push

scrape_configs:
  # BBB Web API
  - job_name: bbb-web
    static_configs:
      - targets: [localhost]
        labels:
          job: bbb
          service: bbb-web
          __path__: /var/log/bigbluebutton/bbb-web.log

  # BBB HTML5 Frontend
  - job_name: bbb-html5
    static_configs:
      - targets: [localhost]
        labels:
          job: bbb
          service: bbb-html5
          __path__: /var/log/bbb-html5/*.log

  # WebRTC SFU (Mediasoup)
  - job_name: bbb-webrtc
    static_configs:
      - targets: [localhost]
        labels:
          job: bbb
          service: bbb-webrtc-sfu
          __path__: /var/log/bbb-webrtc-sfu/*.log

  # FreeSWITCH (Ses)
  - job_name: bbb-freeswitch
    static_configs:
      - targets: [localhost]
        labels:
          job: bbb
          service: freeswitch
          __path__: /opt/freeswitch/var/log/freeswitch/freeswitch.log

  # Nginx
  - job_name: bbb-nginx
    static_configs:
      - targets: [localhost]
        labels:
          job: bbb
          service: nginx
          __path__: /var/log/nginx/*.log

  # Kayıt İşleme (Recording)
  - job_name: bbb-recording
    static_configs:
      - targets: [localhost]
        labels:
          job: bbb
          service: recording
          __path__: /var/log/bigbluebutton/bbb-rap-worker.log

  # PILOS Logları (Docker üzerinden dosya olarak map edilmişse)
  - job_name: pilos
    static_configs:
      - targets: [localhost]
        labels:
          job: pilos
          service: app
          __path__: /opt/pilos/storage/logs/*.log
```

### 2.4 Promtail Systemd Servisi

```bash
sudo nano /etc/systemd/system/promtail.service
```

```ini
[Unit]
Description=Promtail Log Agent
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yml
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now promtail
sudo systemctl status promtail
```

---

## 3. Strateji 2: ELK Stack (Endüstri Standardı)

ELK (Elasticsearch + Logstash + Kibana) daha güçlü arama ve tam metin indeksleme özelliği sunar, ancak daha fazla kaynak gerektirir.

> [!WARNING]
> ELK Stack, minimum **4 GB RAM** (sadece Elasticsearch için 2 GB heap) gerektirir. 8 GB RAM altında Loki tercih edilmelidir.

### 3.1 Filebeat Kurulumu (Her Sunucuda)

Filebeat, Logstash/Elasticsearch'e log gönderen hafif bir göndericidir (shipper).

```bash
# Elastic repo ekle ve Filebeat kur
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update && sudo apt install filebeat -y
```

**Moodle Filebeat yapılandırması:**

```yaml
# /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/nginx/*.log
    fields:
      service: nginx
      platform: moodle

  - type: log
    paths:
      - /var/log/php-fpm/moodle-error.log
    fields:
      service: php-fpm
      platform: moodle
    multiline:
      pattern: '^\['
      negate: true
      match: after

  - type: log
    paths:
      - /var/log/postgresql/*.log
    fields:
      service: postgresql
      platform: moodle

output.elasticsearch:
  hosts: ["http://elk.sirket.com:9200"]
  index: "moodle-logs-%{+yyyy.MM.dd}"
```

```bash
sudo systemctl enable --now filebeat
```

---

## 4. Grafana Alarm Kuralları (Kritik)

Merkezi log sistemine sahip olmak yetmez — hatalar anında bildirilmelidir.

### 4.1 Önerilen Alarm Senaryoları

| Alarm | LogQL Sorgusu (Loki) | Kanal |
| :--- | :--- | :--- |
| PHP Fatal Error | `{service="php-fpm"} \|= "Fatal error"` | Telegram / Slack |
| Nginx 502 Patlaması | `rate({service="nginx"} \|= "502" [5m]) > 10` | E-posta |
| BBB WebRTC Hatası | `{service="bbb-webrtc-sfu"} \|= "error"` | Slack |
| FreeSWITCH Çökmesi | `{service="freeswitch"} \|= "CRASH"` | Telegram |
| PostgreSQL Deadlock | `{service="postgresql"} \|= "deadlock"` | E-posta |
| Redis Bellek Dolması | `{service="redis"} \|= "maxmemory"` | Slack |
| Kayıt İşleme Hatası | `{service="recording"} \|= "ERROR"` | E-posta |

### 4.2 Grafana'da Alarm Oluşturma

```text
Grafana > Alerting > Alert rules > New alert rule
```

1. **Query:** Yukarıdaki LogQL sorgularından birini girin
2. **Condition:** Son 5 dakikada 5'ten fazla eşleşme varsa tetikle
3. **Contact Point:** Telegram bot, Slack webhook veya SMTP e-posta
4. **Notification Policy:** Severity'ye göre gruplama (Critical → anında, Warning → 15 dk)

---

## 5. Log Rotasyonu ve Disk Yönetimi

Loglar kontrol edilmezse disk dolar ve sistem çöker. Her iki platform için de `logrotate` yapılandırması kritiktir.

### 5.1 Moodle Sunucusu

```bash
sudo nano /etc/logrotate.d/moodle
```

```text
/var/log/php-fpm/moodle-error.log
/var/log/nginx/access.log
/var/log/nginx/error.log
{
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        systemctl reload nginx > /dev/null 2>&1 || true
        systemctl reload php8.3-fpm > /dev/null 2>&1 || true
    endscript
}
```

### 5.2 BigBlueButton Sunucusu

```bash
sudo nano /etc/logrotate.d/bbb-custom
```

```text
/var/log/bigbluebutton/*.log
/var/log/bbb-html5/*.log
/var/log/bbb-webrtc-sfu/*.log
{
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}

/opt/freeswitch/var/log/freeswitch/freeswitch.log
{
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

```bash
# Logrotate yapılandırmasını test et
sudo logrotate -d /etc/logrotate.d/moodle
sudo logrotate -d /etc/logrotate.d/bbb-custom
```

---

## 6. Faydalı LogQL Sorguları (Loki/Grafana)

```logql
# Son 1 saatte tüm PHP hataları
{service="php-fpm"} |= "error" | line_format "{{.message}}"

# BBB'de 1020 WebRTC hatası yaşayan kullanıcılar
{service="bbb-webrtc-sfu"} |= "1020"

# PostgreSQL'de 1 saniyeden uzun süren sorgular
{service="postgresql"} |~ "duration: [0-9]{4,}" 

# Nginx'te en çok 502 alan URL'ler
{service="nginx"} |= "502" | pattern `<_> "<method> <url> <_>"` | line_format "{{.url}}"

# BBB toplantı oluşturma hatalarını filtrele
{service="bbb-web"} |= "ERROR" |= "create"

# Son 24 saatte kayıt işleme hatası
{service="recording"} |= "ERROR"
```

---

> [!TIP]
> **Başlangıç önerisi:** Küçük ve orta ölçekli kurulumlar için **Loki + Grafana** yeterlidir ve 1 GB RAM ile çalışır. 10.000+ kullanıcılı kurumsal ortamlarda veya karmaşık arama ihtiyacı varsa **ELK Stack** tercih edilmelidir.
