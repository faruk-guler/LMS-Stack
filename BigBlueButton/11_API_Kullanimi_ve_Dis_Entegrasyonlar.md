# Bölüm 11: API Kullanımı ve Dış Entegrasyonlar (LMS/CMS)

BigBlueButton, kendi başına bir son kullanıcı yönetim paneli sunmaz; bunun yerine güçlü bir REST API (`bbb-web`) barındırır. Greenlight, Moodle, Canvas, WordPress gibi platformlar bu API'yi kullanarak odalar oluşturur, kullanıcıları toplantıya sokar ve kayıtları listeler.

## 11.1 API Checksum (Güvenlik Doğrulaması) Mantığı

BBB API'si ile iletişim kuran her uygulamanın URL sonuna bir `checksum` eklemesi zorunludur. Bu, gönderilen isteğin aradaki biri tarafından değiştirilmediğini kanıtlar.

**Checksum Nasıl Hesaplanır?**
`Kullanılacak_API_Metodu` + `Sorgu_Parametreleri` + `Sizin_Yönetici_Sırrınız (Secret)`

1. Örneğin bir toplantı yaratmak için `create` metodunu çağıracağız.
2. Parametrelerimiz: `name=Matematik&meetingID=mat101`
3. Sırrımız (`bbb-conf --secret` ile öğrendiğimiz): `xyz123abc`
4. Formül: `SHA256("create" + "name=Matematik&meetingID=mat101" + "xyz123abc")`
5. Üretilen SHA256 Hash'i (örneğin `8f1e...`) URL'nin sonuna `&checksum=8f1e...` olarak eklenir.

> [!WARNING]
> BBB 3.0 ile birlikte checksum algoritması **SHA1'den SHA256'ya** değiştirilmiştir. Eski (2.x) sürümlerden yükseltme yapıyorsanız, `bigbluebutton.properties` dosyasında `securitySalt.supportedChecksumAlgorithms=sha256,sha512` ve `checkSumAlgorithmForBreakouts=SHA256` parametrelerini kontrol edin. SHA1 artık varsayılan olarak kabul edilmez.

## 11.2 En Sık Kullanılan API Metotları

| Metot | Ne İşe Yarar? | Örnek Parametreler |
| --- | --- | --- |
| `create` | Yeni bir sanal sınıf oluşturur (Fiziksel başlatmaz, sadece hafızada rezerve eder). | `meetingID`, `name`, `attendeePW`, `moderatorPW`, `record` |
| `join` | Katılımcıyı odaya dahil edecek eşsiz URL'yi üretir. | `meetingID`, `fullName`, `password` (Öğrenci veya Öğretmen şifresi) |
| `end` | Odadaki herkesi atar ve odayı kapatır (Kayıt sürecini başlatır). | `meetingID`, `password` (Sadece öğretmen şifresi kabul edilir) |
| `getMeetingInfo` | O an odada kaç kişi var, kimler konuşuyor XML olarak listeler. | `meetingID`, `password` |
| `getRecordings` | Tamamlanmış MP4/Sunum kayıtlarının linklerini XML olarak döndürür. | `meetingID` |

> [!TIP]
> BBB'ye "API call" yaptığınızda dönen sonuç her zaman **XML** formatındadır. Eğer uygulamanız JSON bekliyorsa ilgili parser/bridge kütüphanelerini (ör: PHP için `bigbluebutton-api-php`) kullanmalısınız.

## 11.3 Moodle ve WordPress (CMS) Entegrasyonları

BBB sunucunuzu ayağa kaldırdıktan sonra bir okula bağlamak istiyorsanız:

1. **Moodle Entegrasyonu:**
   Moodle yöneticisi "Eklentiler" altından `mod_bigbluebuttonbn` (Moodle 4.x'te gömülü gelir) ayarlarına gider.
   - BBB URL: `https://bbb.sirket.com/bigbluebutton/` *(Sonundaki `/` ve `/bigbluebutton/` mutlaka olmalıdır)*
   - Shared Secret: `bbb-conf --secret` çıktıdaki uzun anahtar.

2. **WordPress Entegrasyonu:**
   En popüler eklenti "BigBlueButton API" plugin'idir. Ayarlarına gidip aynı şekilde URL ve Secret Key girilir. Sayfalara veya yazılara Shortcode (örn: `[bigbluebutton meeting="test"]`) ekleyerek odaya katılım butonu çıkarılabilir.

## 11.4 Güvenlik İpuçları

- API Sırrınız (Shared Secret) ele geçirilirse herkes sizin sunucunuz üzerinde oda açıp 1000 saatlik işlemler yaparak sunucu CPU'sunu sömürebilir. Sırrınızı kimseyle paylaşmayın.
- Gerekirse sırrınızı değiştirmek için: `sudo bbb-conf --setsecret yenisifre_buraya`

## 11.5 Webhooks (Anlık Olay Bildirimleri)

BigBlueButton, periyodik olarak API'ye "Toplantı bitti mi? Biri girdi mi?" diye sormanız yerine, bir olay yaşandığında kendi sisteminize anında HTTP POST atılmasını sağlayan **bbb-webhooks** altyapısına sahiptir.

**Kullanım Senaryoları:**

- Bir kullanıcı odaya bağlandığında LMS sisteminize "Katıldı" (UserJoined) sinyali gönderir.
- Bir toplantının mp4 işlenmesi bittiğinde CRM panelinize "Kayıt Hazır" (RecordingReady) bildirimi ve linki ulaştırır.

**Kurulum ve Kullanım:**
Sunucunuzda kurulu gelmemiş olabilir, kurmak için:

```bash
sudo apt install bbb-webhooks
```

Kurulduktan sonra bir "Callback URL (Bildirimlerin gideceği adresiniz)" yaratmak için, `bbb-web` üzerinden `hooks/create` API çağrısı yapmalısınız (Klasik Checksum hesaplayarak). Kayıt işlemi başarılı olduktan sonra yaşanan webRTC olayları dahil tüm aksiyonlar uygulamanıza POST JSON verisi olarak akmaya başlayacaktır.
