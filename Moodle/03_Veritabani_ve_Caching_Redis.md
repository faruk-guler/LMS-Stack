# 03 - Veritabanı ve Caching (Redis & PostgreSQL) İnce Ayarları

Moodle'ın "hız" algısı tamamen veritabanı yanıt süresine ve cache (önbellek) stratejisine bağlıdır. Varsayılan ayarlarla bırakılan bir PostgreSQL, Moodle için yeterince verimli çalışmaz.

> [!NOTE]
> **Redis sunucusunun kurulumu, güvenliği (`requirepass`/`bind`/TLS), `maxmemory`-eviction politikası, kalıcılık ve Sentinel/HA'nın TEK KAYNAĞI:** `Redis/Redis_Rehberi.md`. Bu bölüm yalnızca **Moodle'a özgü** Redis session/MUC yapılandırmasını (`config.php`) içerir.

## 3.1 Neden Redis?

Moodle varsayılan olarak cache verilerini diskte (`moodledata` içinde) saklar. Disk (SSD olsa bile) RAM'den binlerce kat yavaştır.

| Özellik | Diskte Cache | Redis Cache |
| :--- | :--- | :--- |
| Okuma Hızı | ~1–5 ms | < 0.1 ms |
| moodledata I/O Yükü | Yüksek | Sıfır |
| Clustering Desteği | ❌ | ✅ |
| Session Paylaşımı (LB) | ❌ | ✅ |

## 3.2 Redis Session Konfigürasyonu (config.php)

```php
// Redis tabanlı Session yönetimi
$CFG->session_handler_class = '\core\session\redis';
$CFG->session_redis_host     = '127.0.0.1';
$CFG->session_redis_port     = 6379;
$CFG->session_redis_database = 0;
$CFG->session_redis_prefix   = 'mdl_sess_';
$CFG->session_redis_acquire_lock_timeout = 120;
$CFG->session_redis_lock_expire          = 7200;
```

## 3.3 Redis'i Test Etme

```bash
redis-cli ping
# Beklenen yanıt: PONG

redis-cli info keyspace
# Moodle oturumlar başladıktan sonra db0 altında key'ler görünmeli
```

## 3.4 PostgreSQL Performance Tuning

Varsayılan PostgreSQL kurulumu Moodle için çok muhafazakardır. Sunucu kaynaklarını kullanmak için `postgresql.conf` içinde aşağıdaki ayarlar değiştirilmelidir.

> [!WARNING]
> Bu değerler **16 GB RAM** olan bir sunucu için örnek değerlerdir. Farklı donanım için değerleri orantılı şekilde ayarlayın.

```bash
sudo nano /etc/postgresql/17/main/postgresql.conf
```

| Parametre | Varsayılan | Önerilen (16 GB RAM) | Açıklama |
| :--- | :--- | :--- | :--- |
| `shared_buffers` | 128MB | 4GB | RAM'in ~%25'i |
| `work_mem` | 4MB | 64MB | Tek bir sorgu için kullanılabilir bellek |
| `maintenance_work_mem` | 64MB | 512MB | VACUUM, CREATE INDEX için |
| `effective_cache_size` | 4GB | 12GB | OS disk cache tahmini (RAM × 0,75) |
| `max_connections` | 100 | 200 | Moodle'ın ihtiyacına göre |
| `wal_buffers` | -1 (auto) | 64MB | Yazma işlemleri için buffer |
| `checkpoint_completion_target` | 0.9 | 0.9 | I/O yükünü düzleştirir |

```bash
sudo systemctl restart postgresql
```

## 3.5 PHP-OPcache Ayarları

PHP kodlarının her seferinde baştan derlenmemesi için OPcache hayati önem taşır.

```bash
sudo nano /etc/php/8.3/fpm/conf.d/10-opcache.ini
```

```ini
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.revalidate_freq=60    ; Canlı sistemde 60 saniye, geliştirmede 0 yapın
opcache.save_comments=1       ; Moodle annotation'ları için zorunlu
opcache.enable_cli=0
```

## 3.6 Moodle Universal Cache (MUC) Yapılandırması

`Site Administration > Plugins > Caching > Configuration` yoluna gidin:

1. **Redis Instance** oluşturun: `Server` kısmına `127.0.0.1:6379` girin.
2. **Application Cache** için varsayılan sürücüyü Redis olarak ayarlayın.
3. **Session Cache** için Redis'i seçin.
4. **Bağlantıyı Test Edin** — Hata yoksa "Connection successful" mesajı görürsünüz.

> [!IMPORTANT]
> MUC'u Redis'e taşıdıktan sonra 24 saat içinde `redis-cli info memory` ile bellek kullanımını kontrol edin. `maxmemory` limitinizin yeterli olduğundan emin olun.

---

> [!TIP]
> Veritabanı yavaşsa, Moodle Admin panelindeki `Site Administration > Development > Performance info` sayfasından "Veritabanı sorgu sayısı" ve "Veritabanı sorgu süresi" metriklerini açarak hangi sayfaların DB'ye en fazla yük bindirdiğini tespit edebilirsiniz.
