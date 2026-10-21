# 01 - Moodle Mimari ve Bileşenler Deşifresi (Moodle 5.3 LTS)

Moodle **5.1** ile gelen en önemli mimari değişiklik: **`/public` root dizini** konseptidir. Artık web sunucusu, Moodle uygulama klasörünün tamamını değil, yalnızca `public/` alt dizinini dışa açar.

## 1.1 Moodle 5.x Dizin Yapısı (Yeni Mimari)

```text
/var/www/moodle/           ← Uygulama kök dizini (Burada admin/, lib/, mod/ vb. bulunur)
├── public/                ← Web root! (Sadece index.php ve asset'ler buradadır)
│   ├── index.php
│   └── ...                
├── moodledata/            ← Kullanıcı dosyaları (web dışında kalmalı)
└── config.php             ← Kritik yapılandırma dosyası (Kök dizinde kalmalı)
```

> [!IMPORTANT]
> **Moodle 5.1 ile kırıcı değişiklik:** Nginx `root` direktifi artık `/var/www/moodle` değil, `/var/www/moodle/public` dizinine yönlendirilmelidir. 4.x'den 5.x'e geçişte bu yapı değişikliği mutlaka uygulanmalıdır.

## 1.2 Genel Mimari ve İstek Akışı

Bir kullanıcı Moodle'a girdiğinde arka planda şu sıra çalışır:

1. **Nginx** → Gelen HTTPS isteği kabul edilir, `root /var/www/moodle/public`
2. **PHP-FPM** → Nginx, isteği FastCGI üzerinden PHP-FPM havuzuna iletir
3. **Moodle PHP** → İlgili `index.php` veya `view.php` yürütülür
4. **MUC Cache** → Önce Redis/APCu cache kontrol edilir (DB'ye gitmeden önce)
5. **PostgreSQL** → Cache'de yoksa veritabanından veri çekilir
6. **Yanıt** → Derlenmiş HTML kullanıcıya iletilir

> [!NOTE]
> `moodledata` bu akışın dışındadır; dosya **indirme** veya **yükleme** gerektiren isteklerde devreye girer.

## 1.3 Web Katmanı (PHP-FPM & Nginx)

- **PHP-FPM:** `mod_php` devri bitti. PHP-FPM (FastCGI Process Manager), havuz (pool) yönetimi sayesinde yüksek trafikli anlarda istekleri kuyruğa alarak sunucunun çökmesini engeller.
- **Nginx:** Statik dosyaları (JS, CSS, Images) doğrudan sunmak ve PHP isteklerini FPM'e iletmek için en verimli seçenektir.

> [!TIP]
> PHP-FPM'de `pm = dynamic` yerine `pm = ondemand` seçeneği, gece saatlerinde düşük trafikte boş worker süreçlerini otomatik öldürerek RAM tasarrufu sağlar.

## 1.4 Veritabanı Desteği (Moodle 5.3 itibarıyla)

| Veritabanı | Minimum Sürüm | Durum |
| :--- | :--- | :--- |
| PostgreSQL | 17 | ✅ Önerilir |
| MySQL | 8.4 | ✅ Destekleniyor |
| MariaDB | 11.4 | ✅ Destekleniyor |
| Oracle | — | ❌ Moodle 5.x'te kaldırıldı |

> [!WARNING]
> **Oracle Database artık desteklenmiyor.** Moodle 5.x ile Oracle bağdaştırıcısı Moodle çekirdeğinden çıkarılmıştır. Geçiş yapıyorsanız PostgreSQL'e migrasyon planınızı önceden hazırlayın.

## 1.5 Storage Katmanı (Moodledata)

Moodle'ın en kritik ve en çok ihmal edilen parçasıdır.

- **filedir:** Kullanıcı dosyaları içerik hash'iyle (SHA1) saklanır. Aynı dosya 10 derse yüklenmiş olsa bile yalnızca bir kez tutulur.
- **cache / localcache:** Moodle'ın derlediği geçici dosyalar. Bu klasörün okuma/yazma hızı (IOPS) doğrudan site hızını etkiler.
- **trashdir:** Silinen dosyalar 30 gün burada bekler.

> [!WARNING]
> `moodledata` asla web erişimine açık bir dizinde olmamalıdır! Moodle 5.x'te `public/` klasörü web root olduğundan, `moodledata` artık `public/` dışında tutulması daha kolaydır.

## 1.6 Caching (MUC - Moodle Universal Cache)

| Cache Katmanı | Amacı | Önerilen Sürücü |
| :--- | :--- | :--- |
| Application Cache | Herkese ortak veriler (Dil dosyaları) | Redis |
| Session Cache | Kullanıcı oturum bilgileri | Redis |
| Request Cache | Tek sayfa yüklemesinde anlık veriler | Local Memory |

## 1.7 Kritik config.php Parametreleri

```php
$CFG->dbtype    = 'pgsql';                          // PostgreSQL (Önerilen)
$CFG->dbhost    = '127.0.0.1';
$CFG->dbname    = 'moodledb';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'guclu_sifre';
$CFG->wwwroot   = 'https://moodle.sirket.com';      // Zorunlu: HTTPS
$CFG->dataroot  = '/var/www/moodle/moodledata';     // public/ DIŞINDA!
$CFG->dirroot   = '/var/www/moodle';                // Kök dizin (admin/ vb. burada)
$CFG->admin     = 'admin';
```

---

> [!TIP]
> **Özetle:** Moodle 5.x'e geçişte en kritik adım web sunucusu `root` direktifini `public/` klasörüne güncellemektir. Bunu yapmadan 502/404 hataları alırsınız.
