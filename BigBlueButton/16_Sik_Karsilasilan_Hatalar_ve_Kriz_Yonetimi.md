# Bölüm 16: Sık Karşılaşılan Hatalar ve Kriz Yönetimi (Troubleshooting Gelişmiş)

BigBlueButton (BBB) birbirinden bağımsız birçok servisin (Node.js, Ruby, Java, FreeSWITCH, Mediasoup, Nginx) birleşimidir. Kritik hataların büyük bir kısmı, bu servislerin birbirleriyle iletişiminin (mesaj kuyruklarının veya izinlerin) kopmasından meydana gelir.

Bu bölüm, klasik bbb-conf --restart komutunun işe yaramadığı, "Sistem tamamen durdu" krizlerinde hızlıca bakılacak bir SysAdmin acil müdahale rehberidir.

## 16.1 "502 Bad Gateway" Yazan Beyaz Ekran Hatası

Sunucu durup dururken Nginx'in meşhur beyaz 502 Bad Gateway sayfasını vermeye başladı ise, olay Nginx'ten ziyade arkasındaki API'nin çökmesinden kaynaklanır.

**Kriz Tespiti:**

1. `sudo bbb-conf --status` yazın.
2. Eğer listede `bbb-web` durumunda `failed` veya hiç çalışmıyor ibaresi varsa, sistem toplantıları düzenleyen Java uygulamasını ayağa kaldıramamıştır.

**Çözüm Yolları:**

- Bazen API'nin arka planda RAM şişmesi yaşayıp swap yapamaması çökmeye neden olur. `sudo systemctl restart bbb-web` deneyin. Eğer 2 dakika beklemeye rağmen kalkmazsa:
- **Gizli Hata (Var Log):** `/var/log/bigbluebutton/bbb-web.log` dosyasına bakın. "Max Memory" (OOM Killer) hatası görüyorsanız, sunucunuzun RAM'ini artırmanız gerekir.

## 16.2 Kırık Sunumlar (Presentation/PDF Dönüşümünün Takılması)

En kronik problemlerden biridir. Eğitmen PowerPoint (.pptx) veya bir .pdf yüklediğinde ekranda "Conversion is running (Dönüştürülüyor)" yazar fakat 5-10 dakika geçer ve hata verir.

**Mantık:** BBB, ofis dokümanlarını PNG'ye çevirmek için LibreOffice tetikler. Ancak bozuk bir ofis dosyası yüklendiğinde LibreOffice işlemi asılı (zombi process) kalır ve sonraki gelen bütün yüklemelerin kuyruğa takılmasına sebep olur.

**Çözüm (Kuyruğu Temizleme ve Yeniden Başlatma):**

```bash
# Önce servis yöneticisi üzerinden zarif şekilde durdurmayı deneyin
sudo systemctl stop bbb-web

# Sadece bbb-web yanıt vermiyorsa son çare olarak sert sonlandırın
sudo killall -9 soffice.bin
sudo killall -9 bbb-web

# Ardından temp dosyalarındaki kırık/kalıntı yüklemeleri temizleyin
sudo bbb-conf --clean
```

Geri döndüğünde öğretmen sunumu saniyeler içinde tekrar yükleyebilecektir.

## 16.3 Medya Katmanı Limitleri (Yüz Bölünmesi veya Kırmızı Ekran)

Kalabalık bir toplantıda (örn: 30 kameranın açık olduğu) öğretmenlerin yüzleri pikselleniyor (Mozaikleniyor), sesler robotikleşiyor veya "Kamera başlatılamadı" hatası geliyorsa:

1. Sunucunuzun Bant Genişliği (1 Gbit/s vb) yetiyor olabilir ancak CPU'nuza, Kırmızı (Red) frame'leri işleyecek güce sahip olmayabilir.
2. Bu durumda varsayılan Video Codec'ini (H264'ten VP8'e) zorlamak veya kameraların Bitrate oranlarını ciddi şekilde kısmak gerekir.

**Müdahale (Bitrate Kısmak):**

- **Mediasoup** seçiliyse dosya: `/etc/bigbluebutton/bbb-webrtc-sfu/production.yml` içinde `cameraProfiles` bölümünü bulun ve `bitrate` değerlerini (örneğin High: 800'den 400'e, Low: 100'den 50'ye) düşürün; sonra `bbb-webrtc-sfu` servisini yeniden başlatın.
- **LiveKit** (varsayılan) kullanıyorsanız kamera bitrate'leri `bbb-html5.yml` içindeki `webrtc.cameraProfiles` üzerinden kısıtlanır; uygulamak için `bbb-html5` servisini yeniden başlatın.

## 16.4 Sunucu Taşıma (Greenlight / Postgres Veritabanını Kurtarma)

En değerli veri Greenlight Postgres'indedir (kim hangi hesapla kayıt oldu, odalar, davetler). Tam **dump + restore runbook'u** tek kaynakta: `Backups/Yedekleme_Rehberi.md` §4.3 (dump) ve §8.3 (restore).

Kısaca: eski sunucudan `docker exec greenlight-db pg_dump ... > gl_yedek.sql`; temiz bir Ubuntu 24.04'e `bbb-install -g` ile Greenlight'ı sıfırdan kur; yeni DB şemasını temizleyip dump'ı geri yükle ve `docker compose down && up -d`. Böylece hesaplar/odalar dakikalar içinde yeni makinede olur.

## 16.5 TLS (Sertifika) Süresi Dolduğunda Neler Olur?

Eğer `bbb-install.sh` otomatik yenilemeyi başaramazsa (Let's encrypt botu düşerse veya 80 portunuz kapandıysa), SSL süreniz bitiği an tarayıcılar (Chrome/Safari) sisteminizi bloklar. Mikrofon izin panelleri kaybolur.

**Manuel Acil Yenileme Komutu:**

```bash
sudo certbot --nginx renew
# Yenileme başarılı olduktan sonra Nginx'e reload atmayı unutmayın:
sudo systemctl reload nginx
sudo bbb-conf --restart
```

*(Eğer SSL yenilendiği halde kameralar ve ses WebRTC hatası veriyorsa, `bbb-webrtc-sfu` eski sertifikanın cache'ini tutuyor demektir; mutlaka sistemi `bbb-conf --restart` ile tazelemeniz gerekir).*

> Sertifika **yenileme (renewal) ve otomasyonunun tek kaynağı**: Certs — Sertifika Kurulum Rehberi §9 (`Certs/Sertifika_Rehberi.md`).
