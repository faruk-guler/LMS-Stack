# Bölüm 7: İleri Düzey Özelleştirme ve Branding (Markalaştırma)

Öğrenciler veya müşteriler toplantıya bağlandığında sayfanın kurumunuza ait olduğunu hissettirmek için logo, renk ve sistem varsayılan PDF dokümanı değiştirilmelidir.

## 7.1 Varsayılan Sunum (Default PDF) Değiştirme

Sisteme ilk girildiğinde ortada beliren ve HTML5 tahtaya dönüşen o varsayılan PDF'i değiştirmek en sık yapılan işlemdir.

1. **Yeni PDF Hazırlığı:**
Kurumsal temanızla `default.pdf` adında (içinde yönergeler veya logonuz olan) yeni bir PDF yaratın. Boyutunun küçük olmasına (1-2 MB altı) dikkat edin, çünkü hızlı yüklenmesi gerekir.

2. **Sunucuya Yükleme:**
Sunucuda orijinal dosyanın yeri şuradadır:
`/var/www/bigbluebutton-default/default.pdf`

Kendi `default.pdf` dosyanızı bu konuma atıp eskisinin üzerine yazın:

```bash
sudo cp kendi_default.pdf /var/www/bigbluebutton-default/default.pdf
# İzinleri düzeltelim (Nginx erişimi için)
sudo chown -R www-data:www-data /var/www/bigbluebutton-default/
```

1. **Önemli: Apply-Config ile Kalıcı Yapmak**
BBB her `apt upgrade` veya yeniden kurma işlemi gördüğünde bu dosya orijinal BBB reklam dosyasıyla **geri değişecektir**. Bunu engellemek için `apply-config.sh` betiği yaratmalısınız. (Bölüm 4'e atıf).
Kendi pdf'nizi `/etc/bigbluebutton/my-default.pdf` içine koyun. Sonra `nano /etc/bigbluebutton/bbb-conf/apply-config.sh` içine şunu ekleyin:

```bash
#!/bin/bash
cp /etc/bigbluebutton/my-default.pdf /var/www/bigbluebutton-default/default.pdf
chown www-data:www-data /var/www/bigbluebutton-default/default.pdf
```

Ve dosyaya çalışma izni verin: `sudo chmod +x /etc/bigbluebutton/bbb-conf/apply-config.sh`

## 7.2 Logo Değiştirme ve Hoşgeldin Sayfası (Kök Dizin)

Kullanıcılar doğrudan `https://bbb.sirket.com` adresine gittiklerinde BBB'nin kendi varsayılan "Welcome to BigBlueButton" sayfası açılır (Eğer Greenlight yoksa).

Bu sayfayı kapatmak veya kendi ana sayfanıza (ör: şirketin sitesine) yönlendirmek için Nginx `/` router'ı düzenlenmeli:

```bash
sudo nano /etc/nginx/sites-available/bigbluebutton
```

İçinde `location / {` bloğunu bulun. Klasör yolunu değiştirebilir veya doğrudan 301 yönlendirmesi ekleyebilirsiniz:

```nginx
location / {
   return 301 https://sirketiniz.com/;
}
```

## 7.3 HTML5 İstemci (Oda İçi) CSS ve Renk Değişimi

BBB'nin iç renk teması temel `settings.yml` (ve eski `/etc/bigbluebutton/bbb-html5.yml`) dosyası üzerinden bir tema değişkenine (custom theme) bağlanabilir, ancak en güvenilir yol doğrudan `custom.css` kullanmaktır.

Sunucuya özel CSS dosyası eklemek için `bbb-html5.yml` dosyasına harici bir CSS linki eklenebilir:

```yaml
public:
  app:
    customStyleUrl: "https://bbb.sirket.com/ozel_tema.css"
```

Daha sonra `/var/www/bigbluebutton-default/` içerisine `ozel_tema.css` koyarsanız Nginx onu otomatik sunacaktır.

**Örnek (ozel_tema.css):**

```css
/* Üst menü barının rengini kurumsal lacivert (Örn: #002D62) yapmak */
._imports_ui_components_nav_bar__styles__navbar {
   background-color: #002D62 !important;
}

/* "Powered by BigBlueButton" yazısını gizlemek (Tavsiye edilmez ama mümkündür) */
/* ._imports_ui_components_video_provider_video_list__styles__poweredBy { display: none !important; } */
```

> [!NOTE]
> React sınıflarının isimleri BBB sürümlerinde çok küçük de olsa değişebilir (hash yiyebilir). Bu yüzden CSS sınıflarını Chrome DevTools inspect (İncele) ile tespit edip `!important` kuralını kullanarak ezmelisiniz. (Varlığını koruyan sabit id/class'lar bazen değişmektedir).
