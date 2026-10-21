# Bölüm 17: Öğrenme Analitikleri (Learning Dashboard) ve Derin Arayüz Özelleştirmeleri

BigBlueButton sadece bir video konferans ekranı değildir; aynı zamanda akademik performansı ölçen bir eğitim aracıdır. Bu bölümde, sistem yöneticilerinin BigBlueButton'ın en derin kullanıcı arayüzü ayarlarına (Frontend JSON) nasıl müdahale edecekleri ve "Learning Dashboard" sistemini nasıl yönetecekleri anlatılmaktadır.

## 17.1 Öğrenme Analitikleri (Learning Dashboard) Nedir?

Learning Dashboard, moderatörlerin (öğretmenlerin) ders sırasında veya ders bittikten sonra öğrencilerin:

- Ne kadar süre konuştuğunu,
- Hangi anketlere ne cevap verdiğini (Anket skoru),
- Kameralarını açıp açmadığını,
- Dersteki genel "Aktivite Puanını"
görebildikleri güçlü bir modül olup, BBB Node.js backend'ine gömülü ayrı bir servistir.

**SysAdmin Yönergeleri:**
Bu özellik genellikle varsayılan olarak açıktır. "Gizlilik İhlali" nedeniyle bir kurum bu raporlama ekranını tamamen devre dışı bırakmak isterseniz `/etc/bigbluebutton/bbb-html5.yml` dosyasına şu satırları ekleyebilirsiniz:

```yaml
public:
  app:
    learningDashboardEnabled: false    # Analitik tablosunu öğretmenlerden gizler
```

*Ayar değiştirildikten sonra `systemctl restart bbb-html5` komutu çalıştırılmalıdır.*

> [!TIP]
> **Veritabanı Temizliği:** Öğrenci istatistikleri doğrudan MongoDB üzerindeki `learning-dashboard` koleksiyonlarına (tablolarına) yazılır. Veri saklama (KVKK) gereksinimi duymuyorsanız bu istatistik raporlarını MongoDB'den belli periyotlarla silecek cron job'lar yazabilirsiniz.

## 17.2 Tarayıcı Dil Dosyalarına (JSON) Derin Müdahale

Arayüzdeki standart metinleri kurumunuza göre değiştirebilirsiniz. Örneğin arayüzde yazan "Kullanıcı" kelimesini "Öğrenci", veya "Sohbet" yazısını "Canlı Mesajlaşma" yapmak isteyebilirsiniz. Frontend tamamen compile edilmiş (build/paketlenmiş) bir React uygulaması olsa da, çeviri dosyaları sunucuda çıplak JSON formatında durur.

**Dil (Locale) Dosyalarının Konumu:**

```bash
/usr/share/meteor/bundle/programs/web.browser/app/locales/
# Örn: tr.json, en.json, de.json
```

**Değiştirme ve Kalıcı Kılma:**

1. Türkçe dil dosyasını bulun: `nano /usr/share/meteor/bundle/programs/web.browser/app/locales/tr.json`
2. Dosya içerisindeki `app.chat.title`: "Sohbet" gibi "Key:Value" değerlerindeki sağ tarafı editleyin.
3. Node.js backend'i yeniden başlatın: `systemctl restart bbb-html5`

> [!WARNING]
> **Güncelleme Riski:** Sunucu paketlerini güncellediğinizde (`apt upgrade`) bu dosya orijinal BigBlueButton `tr.json` dosyasıyla **ezilir**. Kalıcı çevirileriniz için `apply-config.sh` (Bkz. Bölüm 4) içine bir cp (kopyalama) komutu ekleyerek her restart veya güncelleme sonrasında kendi yedeklediğiniz `custom_tr.json` dosyasının dizine yerleşmesini garanti altına almalısınız.

## 17.3 Yüzlerce Kamerayı Aynı Anda Yönetmek (Pagination ve Layout)

BigBlueButton, bir odaya aynı anda 50, 100 kişinin kamerasını soktuğunuzda HTML5 tarayıcısının sekmesinin çökmesini engellemek için "Sayfalandırılmış Kameralar" (Pagination) özelliği sunar.

Öğretmen, ekranda sadece ilk 16-25 kamerayı görür, aşağıya kaydırdıkça (scroll) veya ok (arrow) tuşlarına bastıkça diğer kullanıcıların kameraları sisteme yüklenir (Lazy load yaklaşımı).

Bu limiti `/etc/bigbluebutton/bbb-html5.yml` içerisinden doğrudan SysAdmin olarak değiştirebilirsiniz:

```yaml
public:
  layout:
    autoSwapLayout: true
  media:
    cameraPagination:
      enabled: true
      desktopPageSizes:   # Masaüstünde aynı ekranda max kamera sayısı
        moderator: 25
        viewer: 16
```

> [!NOTE]
> BBB 3.0 öncesi sürümlerde kamera eşik değerleri `kurento:` YAML anahtarı altında tanımlanıyordu. v3.0 ile bu ayarlar `media:` ve `cameraPagination:` altına taşınmıştır. Eski ayar dosyalarınızı güncellemeyi unutmayın.

## 17.4 Paylaşılan Notlar (BlockNote / Etherpad) ve İhracı

BBB içinde sağ tarafta açılan "Paylaşılan Notlar" editörü, **BBB 4.0 ile birlikte BlockNote**'tur (Etherpad artık kurulmaz/kaldırıldı). Öğrencilerin ortak yazdığı dökümanların, ders bitiminde dışa aktarılması API ve kayıt biçimi (Recording Format) üzerinden yönetilebilir. (Önceki **3.x** hatlarında bu editör NodeJS tabanlı bir **Etherpad** sunucusuydu.)

Eğer harici bir yönetim paneli (Örn. Moodle) kullanıyorsanız `bigbluebutton.properties` üzerinden paylaşılan notların direkt toplantı kaydına (*Recording Format*) dahil edilip edilmeyeceğini veya izleme ekranına metin belgesi olarak yansıtılıp yansıtılmayacağını `allowExportNotes=true` parametresi ile şekillendirebilirsiniz.

---
*Bu ince ayarlar, BigBlueButton'ı standart bir video konferans yazılımından çıkarıp, akademik ihtiyaçlara tamamen boyun eğen ve kurumunuzla bütünleşmiş bir **Sanal Kampüs** altyapısına dönüştürmenizi sağlayacaktır.*
