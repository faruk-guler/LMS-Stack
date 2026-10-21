# Bölüm 5: Toplantı ve Oda Ayarları

BigBlueButton (BBB) kurulumundan sonraki ilk iş, sistemin varsayılan davranışlarını (kameraların kapalı başlaması, anketlerin süresi, varsayılan dil, vb.) kendi senaryonuza (Kurum içi eğitim, Okul, Konferans) göre ayarlamaktır.

BigBlueButton'ın temel ayarları iki ana dosyada yönetilir. Ancak BBB 3.0 sürümleriyle birlikte `/etc/bigbluebutton/bbb-html5.yml` gibi "override" (ezme) dosyaları standart olmuştur.

## 5.1 Temel HTML5 İstemci Ayarları (settings.yml)

BigBlueButton'ın web arayüzü (React/NodeJS Frontend) ile ilgili kararlar `/etc/bigbluebutton/bbb-html5.yml` dosyasından okunur.

> [!CAUTION]
> Asla ama asla `/usr/share/meteor/bundle/programs/server/assets/app/config/settings.yml` (eski sürümler için aynı yerdeki orijinal dosya) dosyasını doğrudan değiştirmeyin. `bbb-conf --restart` yaptığınızda veya sistemi güncellediğinizde yaptığınız tüm ayarlar silinir.
> BBB v3.0 ve sonrasında tüm frontend ayarları `/etc/bigbluebutton/bbb-html5.yml` dosyasında yazılmalıdır (Eğer dosya yoksa sıfırdan oluşturun ve sadece değiştireceğiniz bloğu yazın).

### Örnek bbb-html5.yml Yapılandırmaları

### 1. Katılımcıları (Öğrencileri) Mikrofon ve Kameraları Kapalı Başlatmak

```yaml
public:
  app:
    autoJoin: false    # Otomatik mikrofona katılmayı sorar
  media:
    autoShareWebcam: false # Kamera otomatik paylaşılmaz
  participants:
    everyoneMutedOnJoin: true # Odaya her giren sessizde (muted) başlar.
```

### 2. Veri Tasarrufu Modu (Data Saving) - Kamerayı Zorla Kapatma

Özellikle düşük bant genişliğine sahip ülkelerdeki öğrenciler için sadece öğretmenin kamerası / sunumu gelsin isterseniz:

```yaml
public:
  webrtc:
    cameraProfiles:
      - id: low
        name: Low quality
        default: true
        bitrate: 100
  layout:
    hideVideoForViewers: true # Sadece moderatör - öğretmenin kamerası izleyiciler tarafından görülür.
```

### 3. Uygulamanın Türkçe ile Başlaması (Tarayıcı Dilinden Bağımsız)

```yaml
public:
  app:
    defaultLocale: tr
```

Değişiklik yaptıktan sonra:

```bash
sudo systemctl restart bbb-html5
```

## 5.2 API ve Web (Backend) Ayarları: bigbluebutton.properties

Toplantı ne kadar süre boş kalırsa kapansın? Bir odada toplantı otomatik olarak ne kadar sürede kayda alınsın? Bu gibi backend (sunucu tarafı) API parametrelerini `/etc/bigbluebutton/bigbluebutton.properties` üzerinden düzenlersiniz.

**(veya kökten çözüm: `sudo bbb-conf --restart`)**

## 5.3 Greenlight Yapılandırması (Eğer Kullanılıyorsa)

Eğer LMS (Moodle vb) kullanıyor ve Greenlight ile işiniz yoksa bu kısmı atlayabilirsiniz.

Greenlight, kendi Postgres veritabanına, kendi `.env` ayar dosyasına sahip bir docker bileşenidir.

**Greenlight v3 (Yeni) Ayarları (`~/greenlight-v3/.env`)**

- `BIGBLUEBUTTON_ENDPOINT`: Sizin API url'siniz.
- `BIGBLUEBUTTON_SECRET`: Sizin API Gizli anahtarınız.
- `SMTP_*`: Kayıt olanlara mail gitmesi ve şifre yenileme linkleri çalışması için gerekli SMTP E-posta sunucu bilginiz.
- `ALLOW_MAIL_NOTIFICATIONS`: true (E-posta ile onayı açar).

Değiştirildiğinde Greenlight konteynerini yeniden başlatmak:

```bash
cd ~/greenlight-v3
docker compose down && docker compose up -d
```
