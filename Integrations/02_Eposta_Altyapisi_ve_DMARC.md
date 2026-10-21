# Bölüm 2: E-posta Altyapısı, SMTP ve Gönderilebilirlik (SPF, DKIM, DMARC)

Moodle gibi devasa LMS sistemleri veya BigBlueButton'ın Greenlight arayüzü yüz binlerce e-posta üretir:

- Yeni hesap aktivasyon linkleri
- "Şifremi unuttum" sıfırlama mailleri
- Sınav notu açıklandı, foruma yeni mesaj eklendi bildirimleri
- Yeni toplantı odası açıldığında atılan davet mailleri

Ancak 2024 ve sonrası kurallarda Google (Gmail), Microsoft (Outlook) ve Yahoo spam filtrelerini son derece sıkılaştırmıştır. Sisteminizin e-postaları spam (gereksiz) klasörüne düşüyorsa veya doğrudan sunucuda reddediliyorsa, Moodle'ın bir kusuru yoktur; sunucunuzun "Geri dönük kimlik kanıtı" eksiktir.

## 1. Moodle ve BBB SMTP Yapılandırması

Sistemlerinizin kendi içlerindeki `sendmail/postfix` yerine **kurumsal ve güvenilir bir SMTP hesabı** üzerinden e-posta göndermesi şarttır (Örn: Office365, Google Workspace, SendGrid, Amazon SES).

### 1.1 Moodle SMTP Ayarları

`Site Administration > Server > Email > Outgoing mail configuration`

- **SMTP hosts:** `smtp.office365.com:587` veya `smtp-relay.gmail.com:587`
- **SMTP security:** TLS (Her zaman TLS kullanın; port 465 ise SSL kullanın).
- **SMTP username / password:** Gönderim yetkisi olan servis e-posta hesabı (örnek: no-reply@sirket.com).
- **No-reply address:** `no-reply@sirket.com` (Bu adres, öğrencilerin Moodle'dan gelen e-postalara cevap yazmasını engellemek için "Kimden" alanında görünür).

> [!WARNING]
> Eğer Azure/O365 kullanıyorsanız, Microsoft "SMTP AUTH" özelliğini güvenlik nedeniyle varsayılan olarak `Disabled` (Kapalı) tutar. Exchange admin panelinden `no-reply@sirket.com` hesabı için SMTP AUTH yetkisini "Enabled" olarak aktif etmelisiniz.

### 1.2 Greenlight (BBB) SMTP Ayarları

```bash
cd ~/greenlight-v3
sudo nano .env
```

```env
# Greenlight v3 SMTP Yapılandırması (Resmi sample.env formatı)
SMTP_SENDER_EMAIL=no-reply@sirket.com
SMTP_SENDER_NAME=BigBlueButton
SMTP_SERVER=smtp.office365.com
SMTP_PORT=587
SMTP_DOMAIN=sirket.com
SMTP_USERNAME=no-reply@sirket.com
SMTP_PASSWORD=guclusifrecik
SMTP_AUTH=plain
SMTP_STARTTLS_AUTO=true
```

Yeniden başlatın: `docker compose down && docker compose up -d`

---

## 2. E-postaların Spama Düşmesini Engellemek (DNS Kayıtları)

Moodle'ın veya BBB'nin e-postaları başarıyla gönderdiğini varsayalım. Neden öğrencilerin %30'u "Mail gelmedi" (Junk klasörüne düşme vb.) diyor? Müşteri e-posta sunucusu gelen iletiye bakar: *Gerçekten `no-reply@sirket.com` sunucusundan mı geldi, yoksa başka bir sunucu Moodle üzerinden `sirket.com` adına mailleri mi taklit ediyor (Spoofing)?*

Bunu kanıtlamak için kurumunuzun DNS (Alan Adı Yöneticisi - Cloudflare vb.) yönetici paneline üç sihirli kayıt eklemelisiniz:

### 2.1 SPF (Sender Policy Framework)

Alan adınız (`sirket.com`) adına hangi IP'lerin posta göndermeye yetkili olduğunu açıklar.

- **Tip:** TXT
- **Ad (Host):** `@` (veya sirket.com)
- **Değer:** `v=spf1 include:spf.protection.outlook.com ip4:203.0.113.1 -all`
*(Burada Exchange (Outlook) sunucularına ve `203.0.113.1` IP adresli Moodle sunucumuzun e-posta atmasına izin verdik, diğer her yeri (`-all`) reddettik).*

### 2.2 DKIM (DomainKeys Identified Mail)

Moodle her e-posta gönderdiğinde içine benzersiz kriptografik bir mühür (imza) basar. Alıcı sunucu, bu mührü doğrulamak için DNS'inize bir Açık Anahtar (Public Key) sorar. Eşleşirse "Bu e-posta yolda hiç değiştirilmemiş" der.

- **Nasıl Eklenir?** Bu anahtarı Moodle sunucusu değil, e-postayı üzerinden gönderdiğiniz servis sağlar. (Örneğin Office 365 Exchange panelinden DKIM oluşturup size iki adet CNAME kaydı verir. Bunları DNS'inize girmeniz gerekir).

### 2.3 DMARC (Domain-based Message Authentication)

En katı bekçidir. SPF ve DKIM taramasından geçemeyen yasadışı mühürlere (örneğin adınıza mail atan sahte sunuculara) ne olacağını, e-posta sağlayıcılara söyler.

- **Tip:** TXT
- **Ad:** `_dmarc`
- **Değer:** `v=DMARC1; p=quarantine; rua=mailto:it-rapor@sirket.com;`
*(`p=quarantine`: Eğer mail SPF ve DKIM'den geçemezse doğrudan alıcının SPAM klasörüne at. `p=reject` yaparsanız direkt reddedilir).*

---

## 3. Toplu Bildirim Krizleri

Moodle'ın standart e-posta işleyişi PHP'nin çalışma döngüsüne ve Cron'a bağlıdır. Eğer hoca 5000 öğrencilik bir derse duyuru atarsa, o Cron döngüsü kilitlenebilir.

### 3.1 Moodle Asynchronous Mail Handling

Moodle'ın e-postaları topluca, sayfa yüklemesini dondurmadan arka planda göndermesini sağlamak için:

1. Pano'da `Site Administration > Advanced Features > Enable scheduled tasks` (Ad-hoc görevler) aktif olsun.
2. Formda duyuru atarken hoca "Send immediately" (Anında gönder) kutusunu bile işaretlese Moodle bu eylemi veritabanındaki (DB) `Message Spool` sırasına yazar.
3. Cron, dakikada bir çalışıp bu sıradan (Örn: her seferinde 100 mail) parçalarla mailleri SMTP üzerinden dağıtır.

### 3.2 SMTP Connection Limitleri (Bilinmesi Gereken Hata)

Çoğu kurumsal şirket e-postası (Office 365, Google) **dakikada n adet** veya **günde m adet** (ör: günde maks 10,000) e-posta göndermenize izin verir.

- Eğer Moodle aynı saniyede 500 e-posta basmaya kalkarsa, Office 365 Moodle'ın IP'sini "Spammer" algılayıp "SMTP 421 - Service not available" hatası verebilir.
- **Çözüm:** Bu gibi devasa sistemlerde Moodle bildirimleri için SendGrid, Amazon SES veya Mailgun gibi toplu mail (Transactional Email) API'lerine özel tasarlanmış servisleri kullanın. Moodle SMTP ayarlarında "SMTP Hosts" alanına `smtp.sendgrid.net` girerek limiti kırabilirsiniz.

---

> [!TIP]
> **Kontrol Listesi:** Her şeyin doğru yapıldığından emin olmak için [Mail-Tester.com](https://www.mail-tester.com/) adresine gidin. Oradaki rasgele e-posta adresine Moodle üzerinden bir e-posta ('Test outgoing mail configuration' butonuyla) gönderin ve spama düşme riskinizi 10 üzerinden görün. 10/10 almıyorsanız büyük ihtimalle SPF veya DKIM eksiktir.
