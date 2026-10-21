# 13 - BigBlueButton (BBB) Moodle Entegrasyonu

Moodle ve BigBlueButton arasındaki bağ, modern bir uzaktan eğitim sisteminin en kritik köprüsüdür. Bu bölüm, yalnızca Moodle tarafındaki yapılandırmayı ele alır; BBB kurulumu için bu repodaki `BigBlueButton` dizinini inceleyin.

## 13.1 BBB Eklentisinin Durumu

Moodle **4.0 ve sonrası** (5.x dâhil) sürümlerde `mod_bigbluebuttonbn` eklentisi artık **Moodle çekirdeğine (core) dâhildir**. Normal şartlarda ayrıca kurmanıza gerek yoktur.

> [!NOTE]
> Eğer eklentinin Moodle çekirdeğindeki sürümünden daha yeni bir sürümünü (Beta veya yeni özellikler içeren) manuel olarak kurmak isterseniz aşağıdaki adımları izleyebilirsiniz. Ancak kurumsal üretim (production) sistemlerinde her zaman **Core** sürümde kalınması önerilir.

```bash
# Sadece manuel güncelleme gerektiğinde (İsteğe bağlı):
cd /var/www/moodle/mod
sudo git clone https://github.com/blindsidenetworks/moodle-mod_bigbluebuttonbn.git bigbluebuttonbn
sudo chown -R root:www-data /var/www/moodle/mod/bigbluebuttonbn
sudo -u www-data php8.3 /var/www/moodle/admin/cli/upgrade.php --non-interactive
```

## 13.2 API Konfigürasyonu

BBB sunucunuzun API adresini ve gizli anahtarını (Secret) aşağıya girin:

`Site Administration > Plugins > Activity Modules > BigBlueButton (mod_bigbluebuttonbn)`

| Alan | Değer | Komut (BBB Sunucusunda) |
| :--- | :--- | :--- |
| BBB API Endpoint | `https://bbb.sirket.com/bigbluebutton/` | `bbb-conf --secret` |
| BBB Secret | `xyz123abc...` | `bbb-conf --secret` |

> [!IMPORTANT]
> BBB API Endpoint URL'si mutlaka `/bigbluebutton/` ile **bitmelidir** (sondaki `/` dahil). Eksik olursa tüm API çağrıları başarısız olur.

## 13.3 API Bağlantısını Test Etme

Moodle admin panelinde "Test" butonu yerine CLI ile doğrulama yapın:

```bash
# Checksum hesaplama ve API test (BBB sunucusunda)
SECRET="bbb_secret_key"
CALL="getMeetings"
CHECKSUM=$(echo -n "${CALL}${SECRET}" | sha256sum | awk '{print $1}')
curl -s "https://bbb.sirket.com/bigbluebutton/api/getMeetings?checksum=${CHECKSUM}"
# Yanıtta <returncode>SUCCESS</returncode> görmelisiniz
```

## 13.4 Kayıt (Recording) Yönetimi

BBB üzerinde yapılan kayıtların Moodle dersleri içinde görünmesi için:

1. BBB sunucusunda kayıtların tamamlanmış ve `published` (yayımlanmış) durumda olması gerekir.
2. Moodle Cron'u çalışıyor olmalıdır — Cron her çalıştığında BBB API'sini sorgular.
3. Kayıt listesi Moodle'da hemen görünmüyorsa: `sudo -u www-data php8.3 admin/cli/cron.php`

```bash
# BBB'de kayıt durumlarını kontrol et
SECRET="bbb_secret_key"
CALL="getRecordings"
CHECKSUM=$(echo -n "${CALL}${SECRET}" | sha256sum | awk '{print $1}')
curl -s "https://bbb.sirket.com/bigbluebutton/api/getRecordings?checksum=${CHECKSUM}" | grep -o '<state>[^<]*</state>'
```

## 13.5 Kullanıcı Rolleri ve Eşleşme

| Moodle Rolü | BBB Rolü | İzinler |
| :--- | :--- | :--- |
| Teacher / Editingteacher | Moderator | Toplantıyı başlatır, katılımcıları yönetir |
| Student | Viewer | Sadece katılım |
| Siteadmin | Moderator | Her toplantıya moderatör olarak girer |

Bu eşleştirme `Site Administration > Plugins > Activity modules > BigBlueButton > Manage roles` altından özelleştirilebilir.

## 13.6 Webhooks ile Kayıt Bildirim Optimizasyonu

Moodle varsayılan olarak her Cron çalıştığında BBB API'sini sorgular. Binlerce toplantı için bu verimsizdir.

```bash
# BBB Sunucusunda Webhook kurulumu (BBB sunucusunda çalıştırın)
SECRET="bbb_secret_key"
CALLBACK="https://moodle.sirket.com/mod/bigbluebuttonbn/bbb_broker.php"
# Checksum = apiMetodu + sorguDizgisi (callbackURL=... PARAMETRE ADI DAHIL) + secret
CHECKSUM=$(echo -n "hooks/createcallbackURL=${CALLBACK}${SECRET}" | sha256sum | awk '{print $1}')
curl "https://bbb.sirket.com/bigbluebutton/api/hooks/create?callbackURL=${CALLBACK}&checksum=${CHECKSUM}"
```

## 13.7 Sık Karşılaşılan BBB Bağlantı Sorunları (cURL Engeli)

BigBlueButton veya entegre çalışan Greenlight sunucusuna erişim esnasında API hataları yaşıyorsanız, Moodle'ın varsayılan HTTP güvenlik kalkanı (cURL) lokal isteklerinizi engelliyor olabilir.

**Çözüm:**
`Site Administration > Security > HTTP security > cURL blocked hosts list` menüsüne gidin.
Eğer BBB sunucunuz lokal ağdaysa listedeki `10.0.0.0/8` (veya `192.168.0.0/16`) bloğunu listeden **silin**.
Bu sayede Moodle sunucusunun cURL üzerinden lokal bloktaki BigBlueButton API'sine istek atmasına izin vermiş olursunuz.

---

> [!TIP]
> `Test connection` butonu yeşil yanmıyorsa önce kontrol edilecekler sırasıyla: SSL sertifika geçerliliği → DNS çözünürlüğü → Port 443 erişimi → `SECRET` değerinin doğruluğu. Moodle güvensiz (self-signed SSL) BBB bağlantısına izin vermez.
