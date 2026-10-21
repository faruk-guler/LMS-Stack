# 14 - Sık Karşılaşılan Hatalar ve Debug Teknikleri

Moodle'da bir sorun olduğunda "Beyaz Ekran" (White Screen of Death) veya anlamsız hata kodlarıyla karşılaşmak sistemciler için en sinir bozucu andır. İşte çözüm anahtarları:

## 14.1 Log Konumları — Hızlı Başvuru Tablosu

| Log Dosyası | Konum | İçerik |
| :--- | :--- | :--- |
| Nginx hata logu | `/var/log/nginx/error.log` | 502, 404, permission hataları |
| PHP-FPM logu | `/var/log/php-fpm/moodle-error.log` | PHP fatal, memory, timeout |
| PostgreSQL logu | `/var/log/postgresql/postgresql-17-main.log` | Slow query, deadlock |
| Moodle uygulama logu | `Admin > Reports > Live logs` | Kullanıcı eylemleri |
| Moodle hata logu | `Admin > Reports > Error log` | Moodle iç hataları |
| Systemd Cron | `journalctl -u moodle-cron` | Cron çıktıları |

> [!TIP]
> **Merkezi Log Yönetimi:** Loglarınızı tek bir panelden (Grafana/Loki) izlemek ve alarmlar kurmak için kök dizindeki Merkezi Log Yönetimi (`Integrations/03_Merkezi_Log_Yonetimi.md`) rehberini inceleyin.

## 14.2 Debug Modunu Açmak

```php
// config.php'ye ekleyin (sadece test/geliştirme ortamında!)
@error_reporting(E_ALL);
@ini_set('display_errors', '1');
$CFG->debug        = E_ALL;
$CFG->debugdisplay = 1;
$CFG->debugperformance = true;  // Sayfa altında DB sorgu sayısını gösterir
```

> [!WARNING]
> `debugdisplay = 1` ayarını canlı sistemde asla açık bırakmayın. PHP hata mesajları içinde hassas bilgiler (DB şifresi, dosya yolları) görünebilir.

## 14.3 Memory Limit Hataları

```bash
# Logda "Allowed memory size of ... bytes exhausted" görüyorsanız:
sudo nano /etc/php/8.3/fpm/php.ini

# memory_limit değerini artırın
memory_limit = 1G

# PHP-FPM'i yeniden başlatın
sudo systemctl reload php8.3-fpm
```

## 14.4 Purge Caches (Önbellek Temizleme)

Tema bozuksa, eklenti yükledikten sonra site garip davranıyorsa veya çeviriler görünmüyorsa:

```bash
# Tüm cache'leri temizle (purge_caches.php seçenek/san bayrak kabul etmez)
sudo -u www-data php8.3 /var/www/moodle/admin/cli/purge_caches.php
```

## 14.5 504 Gateway Timeout (PHP-FPM Timeout)

Büyük kursları silerken veya yedek alırken Nginx ile PHP-FPM bağı zaman aşımına düşebilir.

```nginx
# /etc/nginx/sites-available/moodle içinde
fastcgi_read_timeout 600;   # 300'den 600'e artırın
```

```ini
# /etc/php/8.3/fpm/php.ini içinde
max_execution_time = 600
```

```bash
sudo systemctl reload nginx php8.3-fpm
```

## 14.6 "DML Database Exception" Hataları

Genellikle `max_input_vars` değeri yetersiz olduğunda çıkar (özellinkle büyük ölçekli formlarda):

```bash
# Semptom: Çok sayıda seçenek içeren formu kaydederken hata
# Çözüm: php.ini içinde değeri artırın
max_input_vars = 10000

# Deadlock durumunda PostgreSQL logunu inceleyin
sudo tail -100 /var/log/postgresql/postgresql-17-main.log | grep -i "deadlock\|ERROR"
```

## 14.7 Eksik Dosya (File Not Found) Hataları

Veritabanında kayıtlı ancak `moodledata/filedir` içinde bulunmayan dosyalar için:

```bash
# 1) Veritabanı şemasını doğrula: çekirdek, yükseltme sırasında tabloları/sütunları
#    kontrol eder ve eksikse raporlar.
sudo -u www-data php8.3 /var/www/moodle/admin/cli/upgrade.php --non-interactive

# 2) Kayıtlı olduğu hâlde diskte eksik olan dosyaları bul:
#    (contenthash -> filedir/<ilk2>/<sonraki2>/<tum hash> yol mantığıyla)
sudo -u postgres psql moodledb -At -c \
  "SELECT DISTINCT contenthash FROM mdl_files WHERE contenthash <> '' AND contenthash <> 'emptyhash';" \
| while read -r h; do
    f="/var/www/moodle/moodledata/filedir/${h:0:2}/${h:2:2}/${h}"
    [ -f "$f" ] || echo "EKSIK: $h"
  done
```

> [!NOTE]
> Moodle'ın çekirdeğinde tek adımda "eksik dosya onarımı" yapan bir CLI yoktur. Tespit edilen eksik `contenthash` dosyaları için `filedir` ağacını güncel bir yedekten geri yüklemek en güvenli yoldur. Sunum/içerik dağıtım bozukluklarında önce yedek doğrulaması yapın.

## 14.8 Kurulum Sonrası "Site Not Found" Hatası

```bash
# Nginx aktif site yapılandırmasını kontrol et
sudo nginx -T | grep server_name

# Site etkin mi?
ls -la /etc/nginx/sites-enabled/
```

```bash
# Nginx syntax kontrolü
sudo nginx -t && sudo systemctl reload nginx
```

---

> [!TIP]
> Moodle'da bir hata araştırırken her zaman şu sırayı takip edin: **1. Nginx log → 2. PHP-FPM log → 3. PostgreSQL log → 4. Moodle web log**. Arayüzdeki hata mesajı sadece buzdağının görünen kısmıdır.
