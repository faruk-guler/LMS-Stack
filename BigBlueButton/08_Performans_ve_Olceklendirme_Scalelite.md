# Bölüm 8: Performans ve Ölçeklendirme (Scalelite)

BigBlueButton tekil bir sunucudur. Tasarımı gereği 1 odayı (meeting) 2 farklı sunucudaki kaynaklara eşit bölemez (Monolithic Room). Bir oda en fazla 1 sunucunun gücünü kullanabilir. (Önerilen maksimum limit tek odada 100-150 webcamsız katılımcı veya 40-50 webcamli katılımcıdır).

Okullar ve büyük kurumlar binlerce kişiyi aynı anda BBB üzerinde ağırlamak zorundadır. Peki nasıl ölçeklendirilir? Cevap: **Scalelite** (BBB'nin resmî yük dengeleyicisi). Greenlight/PILOS gibi arayüzler tek bir BBB uç noktasına bağlanır; çoklu sunucu dağıtımını Scalelite üstlenir.

## 8.1 Scalelite Nedir?

Scalelite, BigBlueButton'ın yaratıcıları (Blindside Networks) tarafından geliştirilen açık kaynaklı bir Yük Dengeleyici (Load Balancer) mimarisidir.

- Kullanıcı (Öğretmen) Moodle veya Greenlight üzerindeki "Derse Katıl" butonuna basar.
- İstek `scalelite.sirket.com` adresine (Nginx) gelir.
- Scalelite arkasındaki 10 adet farklı BBB sunucusunun CPU/RAM yüküne veya içindeki kişi sayısına bakar.
- En boş olan (örneğin `bbb04.sirket.com`) sunucusuna istek API üzerinden yönlendirilir.
- Odanın tamamı (Tüm giren öğrenciler dahil) `bbb04`'te yaşamını sürdürür.
- İkinci bir öğretmen başka bir ders açtığında, Scalelite onu `bbb07`'ye atar.

> [!CAUTION]
> Scalelite veya herhangi bir BBB Load Balancer, "tek bir odadaki 500 kişiyi 5 farklı sunucuya bölerek" taşımaz. Ölçeklendirme işlemi ODA/TOPLANTI (Meeting) bazındadır. Yani 10 odanız varsa her birini 1 sunucuya dağıtır. Tek odada devasa bir yayın (Örn: 2000 kişi) yapmak isterseniz webiner (Youtube Live/Rtmp) entegrasyonu gerekir. (2026 yılı itibariyle)

## 8.2 Scalelite Mimarisi ve Gereksinimleri

Scalelite kurulumu için klasik BBB mimarisinden farklı bir topoloji gerekir:

1. **Önyüz / LMS:** Moodle, Canvas, Greenlight vb. (Kullanıcı giriş ekranı)
2. **Scalelite Sunucusu (veya Cluster'ı):** Nginx ve Ruby/Postgres tabanlı Docker ile çalışan LB.
3. **Worker BBB'ler:** Çok sayıda (Örn: bbb01, bbb02, bbb03) arkada duran "İşçi" sunucular. (Bu sunucular dışarıdan IP tabanlı olarak direkt erişime kapalı olmasa da, kullanıcıların FQDN'ini bilmesine gerek yoktur).
4. **Ortak Depolama (NFS Server veya AWS S3):**
   - Öğretmen bbb04'te ders anlattı ve ders kaydedildi diyelim. Öğretmen ertesi gün bbb01'e denk gelirse dünkü kaydını nasıl görecek?
   - Bütün worker BBB'ler kayıt bittiğinde MP4 / Presentation dosyalarını NFS (Ortak Ağ Sürücüsü) alanına kopyalar (`rsync` veya `scp` mantığıyla).
   - Kullanıcı Moodle'dan "Dünkü Dersi İzle" dediğinde, Scalelite o kaydı doğrudan NFS'nin üzerinden (Nginx alias ile) oynatır.

## 8.3 Arayüz Ölçeklemesi: Greenlight vs PILOS

Greenlight v3 **tek bir BBB uç noktasına** (`BIGBLUEBUTTON_ENDPOINT` + `BIGBLUEBUTTON_SECRET`) bağlanır; içinde çoklu sunucu yük dengeleme **yoktur**. Greenlight'ı ölçeklendirmek için önüne **Scalelite** koyun — Greenlight yalnızca Scalelite uç noktasını bilir, worker'ları değil.

Arayüz düzeyinde **çoklu BBB sunucusunu kendi içinde** dengelemek (Scalelite kurmadan) istiyorsanız, bunu destekleyen resmî seçenek Greenlight değil **PILOS**'tur (bkz. `BigBlueButton/13_PILOS_Alternatif_Arayuz.md`). Merkezi kayıt yönetimi (NFS/S3) her durumda Scalelite veya PILOS tarafında ayrıca yapılandırılır.

## 8.4 Performans Sıkılaştırması (Kamera Limitleri)

LB dışında, elinizdeki tek sunucunun performansını optimize etmenin en iyi yolu kamera limitlerini kısıtlamaktır.

> [!IMPORTANT]
> `maxWebcams` parametresi `/etc/bigbluebutton/bigbluebutton.properties` dosyasında yer alır. Ancak `webcamsOnlyForModerator` gibi kayıt ve görüntüleme davranışı ayarları `/etc/bigbluebutton/bbb-html5.yml` dosyasında tanımlanmalıdır.

**`bigbluebutton.properties` içinde (Backend — bbb-web):**

- `maxWebcams=5` : Sistemde aynı anda en fazla 5 kamera açılmasına izin ver.

Değişiklik sonrası: `sudo systemctl restart bbb-web`

**`bbb-html5.yml` içinde (Frontend — bbb-html5):**

```yaml
public:
  media:
    # Sadece moderatörlerin kamerası görünsün, katılımcılar birbirini göremez
    webcamsOnlyForModerator: true
```

Değişiklik sonrası: `sudo systemctl restart bbb-html5`
