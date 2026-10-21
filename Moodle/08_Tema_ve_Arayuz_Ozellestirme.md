# 08 - Tema ve Arayüz Özelleştirme (Sistemci Bakışı)

Bir sistem yöneticisinin tema ile ne işi olur? Moodle temaları (Örn: Boost) sadece renk değildir; derlenmiş SCSS dosyaları, cachelenmiş ikonlar ve performansı etkileyen JS yükleridir.

## 8.1 Boost Child Theme Oluşturma

Moodle'ın varsayılan teması olan **Boost** üzerinde doğrudan değişiklik yapmak, her güncellemede değişikliklerinizin silinmesine neden olur. Doğru yöntem bir "Child Theme" oluşturmaktır.

```bash
# Child theme dizinini oluşturun
sudo mkdir -p /var/www/moodle/theme/kurumtema

# Zorunlu dosyaları oluşturun
sudo nano /var/www/moodle/theme/kurumtema/config.php
```

```php
<?php
// Moodle Child Theme - config.php
defined('MOODLE_INTERNAL') || die();

$THEME->name = 'kurumtema';
$THEME->fullname = 'Kurum Teması';
$THEME->parents = ['boost'];   // Boost'u temel al
$THEME->enable_dock = false;
$THEME->yuicssmodules = [];
$THEME->rendererfactory = 'theme_overridden_renderer_factory';
$THEME->scss = function($theme) {
    return theme_kurumtema_get_main_scss_content($theme);
};
```

```bash
# SCSS özelleştirme dosyası
sudo mkdir -p /var/www/moodle/theme/kurumtema/scss
sudo nano /var/www/moodle/theme/kurumtema/scss/kurumtema.scss
```

```scss
// Bootstrap ve Boost değişkenlerini override edin
$primary: #1a5897;        // Kurumsal mavi
$secondary: #e8f0fe;
$font-family-base: 'Inter', sans-serif;
$navbar-header-color: #ffffff;

// Boost SCSS'ini dahil et
@import "moodle";
```

## 8.2 SCSS Derleme ve Cache Yönetimi

```bash
# Tema cache'ini temizle (tema değişikliğinden sonra zorunlu)
# Not: purge_caches.php bir seçenek/san bayrak ("--type") kabul etmez; tüm MUC önbelleğini temizler.
sudo -u www-data php8.3 /var/www/moodle/admin/cli/purge_caches.php
```

> [!WARNING]
> **Theme Designer Mode** geliştirme sırasında açın (`Site Admin > Appearance > Themes > Theme settings > Theme designer mode`), ancak canlı sistemde mutlaka kapalı tutun. Her sayfa yüklemesinde SCSS baştan derlenmesi ciddi performans kaybına yol açar.

## 8.3 Custom CSS Enjeksiyonu (Hızlı Yöntem)

Tüm tema altyapısını değiştirmek yerine sadece kurumsal renkleri uygulamak için:

`Site Administration > Appearance > Themes > Boost > Advanced Settings > Raw SCSS`

```scss
// Ana menü rengi
.navbar {
    background-color: #1a5897 !important;
}

// Giriş sayfası başlık rengi
#page-login-index h2 {
    color: #1a5897;
}

// Buton rengi
.btn-primary {
    background-color: #1a5897;
    border-color: #154880;
}
```

> [!NOTE]
> Bu alanda yaptığınız değişiklikler `moodledata/lang` dizinine değil, PostgreSQL veritabanına kaydedilir. Moodle güncellemelerinde **silinmez**.

## 8.4 Dil Paketleri ve Yerelleştirme

```bash
# Türkçe dil paketini CLI ile yükle
sudo -u www-data php8.3 /var/www/moodle/admin/tool/langimport/cli/langimport.php --lang=tr
```

> [!NOTE]
> Arayüzden (GUI) tema veya eklenti yükleme adımları:
>
> 1. `Site Yönetimi > Eklentiler > Eklenti yükle` menüsüne gidin.
> 2. Hazırladığınız veya indirdiğiniz tema/eklenti dosyasını (`.zip`) yükleyin.
> 3. Temayı aktif etmek için: `Site Yönetimi > Görünüm > Temalar > Tema seçici` yolunu izleyip aktif edin.

Belirli terimleri kuruma özel değiştirmek için (Örn: "Ders" → "Eğitim"):

`Site Administration > Language > Language customization > tr`

Yaptığınız değişiklikler `moodledata/lang/tr_local/` altına kaydedilir ve güncellemelerde korunur.

## 8.5 Logo ve Favicon Yönetimi

```text
Site Administration > Appearance > Logos
```

- **Site logo:** Giriş sayfasında ve navbar'da kullanılır.
- **Site compact logo:** Küçük header alanları için (ör. mobil).
- **Site icon (favicon):** Tarayıcı sekmesi ikonu.

| Dosya | Minimum Boyut | Önerilen Format |
| :--- | :--- | :--- |
| Site logo | 200 × 50 px | SVG veya PNG (şeffaf arkaplan) |
| Compact logo | 60 × 60 px | SVG veya PNG |
| Favicon | 32 × 32 px | PNG veya ICO |

---

> [!TIP]
> Temada bir hata yaptınız ve site açılmıyor mu? `config.php` dosyasına geçici olarak `$CFG->theme = 'boost';` satırını ekleyerek yedek temaya geçebilirsiniz. Ardından `purge_caches.php` çalıştırın.
