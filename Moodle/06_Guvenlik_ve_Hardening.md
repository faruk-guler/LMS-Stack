# 06 - Güvenlik ve Hardening (Sıkılaştırma)

Moodle, öğrenci verilerini barındırdığı için saldırganlar için cazip bir hedeftir. Standart bir kurulum birçok güvenlik açığını kapalı bırakır.

> [!NOTE]
> **Platform-geneli sertleştirme temeli** (OS/SSH, UFW firewall posture, Fail2Ban, sysctl, dosya izin modeli, HTTP güvenlik başlıkları, WAF/CDN) tek kaynakta: `Security/Guvenlik_Rehberi.md`. Bu bölüm yalnızca **Moodle'a özgü** ayarları (config.php hardening, moodledata/kod izinleri, dahili güvenlik raporu) içerir.

## 6.1 Dosya İzinleri (Permission Hardening)

```bash
# Moodle kod dizini (Kök): root sahipliği, www-data sadece okuma
sudo chown -R root:www-data /var/www/moodle
sudo find /var/www/moodle -type f -exec chmod 640 {} \;
sudo find /var/www/moodle -type d -exec chmod 750 {} \;
# public dizinine Nginx erişimi için izin
sudo chmod -R 755 /var/www/moodle/public

# config.php en kısıtlı izinlerde olmalı (kod kökünde, public dışında)
sudo chmod 640 /var/www/moodle/config.php

# moodledata: sadece www-data yazabilmeli
sudo chown -R www-data:www-data /var/www/moodle/moodledata
sudo chmod -R 750 /var/www/moodle/moodledata
```

> [!WARNING]
> Eklenti yüklemek veya Moodle güncellemek için geçici olarak `www-data`'ya yazma yetkisi vermeniz gerekebilir. İşlem bittikten sonra derhal geri alın.

## 6.2 HTTPS ve Çerez Güvenliği (config.php)

`config.php` içinde mutlaka `$CFG->wwwroot` adresini `https://` ile başlatın:

```php
$CFG->wwwroot = 'https://moodle.sirket.com';

// Moodle'ın çerezleri sadece HTTPS üzerinden göndermesi
$CFG->cookiesecure = true;

// Çerezin sadece HTTP tarafından okunabilmesi (JavaScript erişimini engeller)
$CFG->cookiehttponly = true;
```

Nginx vhost güvenlik başlıkları (**HSTS / X-Frame-Options / nosniff / Referrer-Policy**) tek kaynakta: `Security/Guvenlik_Rehberi.md` §8. TLS/sertifika üretimi ve kurulumu: `Certs/Sertifika_Rehberi.md`.

## 6.3 Güvenlik Duvarı (UFW)

Sunucunun UFW posture'u (default deny incoming; 80/443; PostgreSQL/Redis **dışa kapalı**; yönetim panelleri için IP allowlist) tek kaynakta: `Security/Guvenlik_Rehberi.md` §4 ve §10.

## 6.4 config.php Hardening Parametreleri

```php
// Hata mesajlarını web arayüzünde gösterme (canlı sistemde kapalı olmalı)
$CFG->debug = 0;
$CFG->debugdisplay = 0;

// Moodle'ın HTTPS protokolünü zorla
// Nginx SSL'i sonlandırıyorsa (SSL termination / reverse proxy) true yapın:
$CFG->sslproxy = true;

// SMTP aracılığıyla e-posta: SSL bağlantısı
$CFG->smtpsecure = 'tls';

// Belirli bir IP'den erişim yoksa oturumu sonlandır
// (Mobil kullanıcılar için dikkatli kullanın)
// $CFG->sessioncookiepath = '/';
```

## 6.5 Moodle Yerleşik Güvenlik Denetimi

```text
Site Administration > Reports > Security checks
```

Buradaki **tüm kırmızı uyarıları yeşile çevirmek** bir sistem yöneticisinin asli görevidir.

| Kontrol | Önem | Sık Görülen Neden |
| :--- | :--- | :--- |
| Moodledata web'de erişilebilir | 🔴 Kritik | Dizin yanlış konumda |
| OPcache etkin değil | 🔴 Yüksek | php.ini ayarı eksik |
| Cron son 1 günde çalışmamış | 🔴 Yüksek | crontab veya timer yok |
| `wwwroot` HTTP kullanıyor (SSL yok) | 🟡 Orta | `$CFG->wwwroot` https değil |
| Hata ayıklama ekranda | 🟡 Orta | debug=1 bırakılmış |
| open_basedir kullanılmıyor | ⚪ Düşük | İsteğe bağlı |

---

> [!TIP]
> `config.php` dosyasını asla Git reposuna pushlamayın. İçindeki `dbpass` ve `passwordsaltmain` değerlerini hiç kimseyle paylaşmayın. Şifreyi güncellemek gerekirse `admin/cli/reset_password.php` CLI aracını kullanın.
