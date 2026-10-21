# 11 - Mobil Uygulama ve Push Bildirimleri

Moodle Mobile, öğrencilerin en çok kullandığı arayüzdür. Ancak düzgün yapılandırılmazsa "Siteye bağlanamıyor" hataları ile destek taleplerini kilitler.

## 11.1 Web Services Aktivasyonu

Moodle Mobile uygulamasının çalışması için Web Servisleri ve REST protokolü etkinleştirilmelidir.

```text
Site Administration > Advanced Features > Enable web services ✅
Site Administration > Plugins > Web services > Manage protocols > REST protocol ✅
Site Administration > Mobile app > Enable mobile web service ✅
```

Config.php üzerinden de zorlayabilirsiniz:

```php
// Mobil uygulamanın bağlanabilmesi için
$CFG->enablewebservices = 1;
$CFG->enablemobilewebservice = 1;
```

## 11.2 Mobil Bağlantı Testi

```bash
# API endpoint'ini test et (geçerli bir token ile site_info döndürmeli)
curl -X GET "https://moodle.sirket.com/webservice/rest/server.php?wsfunction=core_webservice_get_site_info&wstoken=TOKEN&moodlewsrestformat=json"
```

> [!NOTE]
> Uygulama belirteci (token) CLI ile üretilmez. Bir kullanıcı için token almak üzere: `Site Administration > Users > Accounts > Tokens` (veya kullanıcının kendi `Preferences > Security keys > Create a token`) yolunu kullanın. Mobil uygulama ise kendi kimlik doğrulamasını **OAuth2** üzerinden yapar (`Site Administration > OAuth2 > Services`).

> [!NOTE]
> Mobil uygulama ilk bağlandığında `core_webservice_get_site_info` çağrısını yapar. Bu çağrı başarısız oluyorsa geçersiz site URL'si veya SSL sorunudur — tarayıcıda siteyi açarak SSL sertifikasının geçerli olduğunu doğrulayın.

## 11.3 Mobil Uygulama Görünümü Özelleştirme

```text
Site Administration > Mobile app > Mobile appearance
```

| Ayar | Açıklama | Örnek Değer |
| :--- | :--- | :--- |
| Main colour | Ana renk | #1a5897 |
| Link colour | Bağlantı rengi | #0d47a1 |
| Feature disabled | Gizlenecek özellikler | `NoDelegate_CoreCoursesDashboard` |
| Forced language | Zorunlu dil | tr |

## 11.4 Airnotifier ve Push Bildirimleri

Moodle varsayılan olarak bildirimleri **Moodle HQ'nun Airnotifier sunucuları** üzerinden gönderir. Bu ücretsizdir ancak özelleştirilemez.

```text
Site Administration > Messaging > Airnotifier settings
```

Varsayılan değerleri bırakırsanız Moodle HQ altyapısı kullanılır. Kendi uygulamanızı App Store/Play Store'da yayınlamak istiyorsanız kendi Airnotifier sunucunuzu kurmanız ve Firebase (FCM) / Apple (APNs) anahtarlarını girmeniz gerekir.

## 11.5 Offline Çalışma ve Senkronizasyon

Moodle Mobile'ın en önemli özelliği, içerikleri **indirip çevrimdışı** kullanabilmesidir.

> [!WARNING]
> SCORM paketlerinin mobil uygulamada çalışması için SCORM dosyalarınızın **mutlak yol** (absolute paths) içermediğinden emin olun. Bazı hazır SCORM araçları `file:///C:/...` gibi yerel yollar kullanır ve bu mobilde çalışmaz.

## 11.6 Mobil Hatalarda Debug

```bash
# Mobil uygulamanın oluşturduğu tokenleri ve aktivitelerini logla
# config.php içine ekle (sadece test ortamında!)
$CFG->debugwebservices = true;

# PHP-FPM hatalarını izle
sudo tail -f /var/log/php-fpm/moodle-error.log | grep -i "webservice\|mobile\|token"
```

---

> [!TIP]
> Öğrencilere uygulamayı tanıtmadan önce kurulumun doğru olup olmadığını **gerçek Moodle Mobile uygulamasıyla** (veya masaüstündeki MoodleNet/QR eşleştirme ile) test edin. Uygulama ilk bağlandığında `core_webservice_get_site_info` çağrısını yapar; bu başarısızsa site URL'si veya SSL yapılandırmasını kontrol edin.
