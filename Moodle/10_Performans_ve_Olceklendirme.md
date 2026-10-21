# 10 - Performans ve Ölçeklendirme (Horizontal Scaling)

Tek bir sunucunun yetmediği noktada Moodle'ı yatayda (Multiple Web Nodes) ölçeklendirmek gerekir. Bu işlem Moodle'ı "Stateless" hale getirmeyi zorunlu kılar.

## 10.1 Tek Sunuculu Yapıda Performans Optimizasyonu

Ölçeklendirmeden önce tek sunucuyu optimize edin:

| Ayar | Nerede | Etki |
| :--- | :--- | :--- |
| Redis Session Cache | config.php | Disk I/O → RAM |
| PHP OPcache | php.ini | Her yüklemede derlemeyi engeller |
| Nginx static file cache | nginx.conf | PHP-FPM'i devre dışı bırakır |
| PostgreSQL tuning | postgresql.conf | Sorgu yanıt süresini düşürür |
| Moodle MUC → Redis | Admin panel | DB yükünü azaltır |

## 10.2 Nginx Upstream Load Balancer Yapılandırması

```nginx
# /etc/nginx/conf.d/moodle-upstream.conf
upstream moodle_backend {
    # Sticky Session: Redis session kullanıyorsanız ip_hash'e gerek yoktur
    # ip_hash;

    server 192.168.1.10:443 weight=1;  # Web Node 1
    server 192.168.1.11:443 weight=1;  # Web Node 2
    server 192.168.1.12:443 weight=1;  # Web Node 3

    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name moodle.sirket.com;

    ssl_certificate     /etc/letsencrypt/live/moodle.sirket.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/moodle.sirket.com/privkey.pem;

    location / {
        proxy_pass https://moodle_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300;
    }
}
```

> [!IMPORTANT]
> Multi-node yapıda **Redis Session** zorunludur. `ip_hash` (sticky sessions) kullanıyorsanız Redis'e gerek yok; ancak bir node çöktüğünde tüm o node'daki kullanıcılar oturum kaybeder. Redis tercih edilir.

## 10.3 Paylaşılan Depolama (Shared moodledata)

Birden fazla web node aynı `moodledata` dizinine erişmek zorundadır.

```bash
# NFS Server kurulumu (depolama sunucusunda)
sudo apt install nfs-kernel-server -y
echo "/var/www/moodle/moodledata  192.168.1.0/24(rw,sync,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -ra

# NFS Client kurulumu (her web node'da)
sudo apt install nfs-common -y
sudo mkdir -p /var/www/moodle/moodledata
echo "192.168.1.5:/var/www/moodle/moodledata  /var/www/moodle/moodledata  nfs  defaults,_netdev  0  0" | sudo tee -a /etc/fstab
sudo mount -a
```

## 10.4 Web Node config.php Ayarları (Multi-Node)

```php
// Her web node aynı config.php'yi kullanmalı
// Veya ortak bir NFS noktasından config.php okunabilir

// Reverse proxy arkasındaysa HTTPS algılaması
$CFG->sslproxy = true;

// Redis Cluster (birden fazla Redis sunucusu var ise)
$CFG->session_redis_host = '192.168.1.20';  // Ortak Redis sunucusu

// Hangi node olduğunu loglara yazdır (debug için faydalı)
$CFG->logsql = false;
```

## 10.5 PostgreSQL Primary-Replica (Okuma Ölçekleme)

> [!IMPORTANT]
> **Moodle çekirdeği, sorguları otomatik olarak replica'lara yönlendiren yerel bir "read/write splitting" mekanizması sunmaz.** `$CFG->dboptions` içinde böyle bir `readonly`/`instance` anahtarı yoktur. Tek bir `$CFG->dbhost` üzerinden bağlanılır.

Okuma yükünü replica'lara dağıtmak için **veri tabanı katmanında** bir çözüme ihtiyaç vardır:

- **Pgpool-II** veya **Patroni + HAProxy** gibi bir proxy, SELECT'i replica'lara, yazmayı primary'e yönlendirir. Moodle bunu bilmeden tek bir uç noktaya (proxy VIP) bağlanır.
- **`$CFG->dboptions`** yalnızca `dbport`, `dbsocket`, `persistent`, `connecttimeout`, `dbsslmode` gibi bağlantı seçenekleri içerir.

```php
// config.php — ör. bir Pgpool VIP'sine bağlanma (uygulama tarafı tek nokta)
$CFG->dbhost    = 'pgpoolVIP';   // yazma primary'e, okuma replica'lara proxy karar verir
$CFG->dboptions = [
    'dbport'         => 9999,     // Pgpool varsayılan portu
    'connecttimeout' => 5,
    // NOT: 'readonly' / 'instance' gibi bir anahtar YOKTUR; okuma/yazma bölmesini proxy (Pgpool) yapar
];
```

> [!NOTE]
> Replication gecikmesi (replication lag) yüksekse, yazmadan hemen sonra okunan sayfalar bayat veri görebilir. Pgpool/HAProxy tarafında "write consistency" ayarlarını mutlaka yapılandırın.

---

> [!WARNING]
> Load Balancing yapılandırmasında sunucuların saatlerinin (NTP ile) milisaniye hassasiyetinde senkronize olduğundan emin olun. Saat farklılıkları "Invalid Token" ve oturum tutarsızlığı hatalarına yol açar: `sudo timedatectl set-ntp true`
