# Bölüm 1: Kurumsal Kimlik Yönetimi ve SSO (LDAP / SAML2 / OAuth2)

Moodle ve BigBlueButton'ı (BBB) binlerce kullanıcılı kurumsal bir ortama (Üniversite, Holding, Kamu) entegre ederken manuel kullanıcı açmak operasyonel bir yüktür. Kullanıcıların mevcut e-posta adresleri veya şirket şifreleri ile her iki sisteme giriş yapabilmesi (Single Sign-On - SSO) bir zorunluluktur.

Bu belgede, Moodle ve Greenlight (BBB arayüzü) sistemlerini Microsoft Entra ID (eski adıyla Azure AD), Google Workspace veya şirket içi Active Directory (LDAP) gibi merkezi kimlik sağlayıcılarına nasıl bağlayacağınızı inceleyeceğiz.

## 1. Active Directory (LDAP) Entegrasyonu

Şirketinizin kendi sunucularında çalışan klasik bir Active Directory varsa, en hızlı entegrasyon yöntemi LDAP'tır.

### 1.1 Moodle LDAP Yapılandırması

Moodle yerleşik bir LDAP eklentisi ile gelir.

1. **Eklentiyi Aktifleştirin:** `Site Administration > Plugins > Authentication > Manage authentication` altından `LDAP server`'ı (Göz ikonuna tıklayarak) aktif edin ve ayarlarına girin.
2. **Sunucu Ayarları:**
   * **Host URI:** `ldap://dc01.sirket.local` veya SSL kullanıyorsanız `ldaps://dc01.sirket.local` (Önerilen).
   * **Version:** 3
   * **TLS Tipi:** Eğer `ldap://` kullanıp güvenlik istiyorsanız "Use TLS" = Yes.
3. **Bağlantı Ayarları (Bind):**
   * **Distinguished name:** `CN=MoodleRead,OU=ServiceAccounts,DC=sirket,DC=local` (Sadece okuma yetkisi olan özel bir servis hesabı (Service Account) açın).
   * **Password:** Servis hesabının şifresi.
4. **Kullanıcı Arama Ayarları:**
   * **Contexts:** `OU=Ogrenciler,DC=sirket,DC=local; OU=Akademisyenler,DC=sirket,DC=local` (Moodle sadece bu OU'lar içinde kullanıcı arar).
   * **Search subcontexts:** Yes
   * **User attribute:** `sAMAccountName` (Active Directory için standart kullanıcı adı niteliği).
5. **Data Mapping (Veri Eşleme):**
   * AD'deki kullanıcının adı, soyadı, e-posta gibi bilgilerini Moodle alanlarına eşleyin.
   * `First name` → `givenName`
   * `Surname` → `sn`
   * `Email address` → `mail`

### 1.2 Greenlight v3 ve LDAP (ÖNEMLİ MİMARİ DEĞİŞİKLİĞİ)

Greenlight v2 sürümünde doğrudan `.env` dosyasına LDAP parametreleri (`LDAP_SERVER`, `LDAP_UID` vb.) girilerek Active Directory bağlantısı yapılabiliyordu. Ancak **Greenlight v3 ile birlikte doğrudan LDAP desteği kaldırılmıştır**.

Bunun yerine, endüstri standardı olan **OpenID Connect (OIDC)** kullanımına geçilmiştir. Greenlight v3 ortamında kullanıcılarınızı LDAP üzerinden doğrulamak için araya bir **Kimlik Sağlayıcı (IdP - Örn: Keycloak)** kurmanız zorunludur.

**Yeni Mimari Akışı:**

1. Kullanıcı Greenlight v3'e girer ve "Giriş Yap" butonuna basar.
2. Greenlight, kullanıcıyı Keycloak (IdP) sunucusuna yönlendirir.
3. Keycloak, arka planda (User Federation özelliği ile) Active Directory (LDAP) sunucunuzla konuşarak kullanıcının bilgilerini doğrular.
4. Keycloak onayladığı kullanıcıyı bir OIDC bileti (Token) ile Greenlight'a geri gönderir.

Bu kurulumun detayları, BBB dokümantasyonundaki ilgili "Greenlight v3 İleri Düzey Yönetim" bölümünde anlatılmıştır. Moodle kullanıyorsanız Greenlight'a ihtiyacınız yoktur; Moodle zaten doğrudan LDAP destekler.

### 1.3 PILOS ve Doğrudan LDAP Desteği

Greenlight v3'ün aksine **PILOS**, Active Directory (LDAP) bağlantısını **doğrudan** desteklemeye devam eder. Araya Keycloak gibi ek bir IdP kurma zorunluluğu yoktur.

PILOS `.env` dosyasında şu parametrelerle LDAP aktif edilebilir:

* `LDAP_ENABLED=true`
* `LDAP_HOST=dc01.sirket.local`
* `LDAP_BASE_DN="OU=Kullanicilar,DC=sirket,DC=local"`
* `LDAP_USER_ATTRIBUTE=sAMAccountName`

> [!NOTE]
> PILOS, nitelik eşleştirme (mapping) ve Regex tabanlı rol atamaları için `.env` dışında gelişmiş bir **JSON yapılandırma** sistemi kullanır. Bu sayede LDAP gruplarını doğrudan PILOS rollerine otomatik olarak bağlayabilirsiniz. Detaylar için: PILOS Bölüm 13 - JSON Mapping (`BigBlueButton/13_PILOS_Alternatif_Arayuz.md`)

---

## 2. SAML2 Entegrasyonu (Modern Kurumsal Çözüm)

LDAP eski, şifreyi ağ üzerinden (şifrelenmiş olsa bile) gönderen bir protokoldür. Günümüzde **Microsoft Entra ID (Azure AD)** veya **Keycloak** kullanılıyorsa, endüstri standardı SAML2 (Security Assertion Markup Language) olmalıdır.

SAML2 mimarisinde kullanıcı şifresini asla Moodle'a girmez; Moodle, kullanıcıyı Microsoft/Google giriş sayfasına yönlendirir, doğrulama orada yapılır ve Moodle'a "Bu kullanıcı doğrulandı, e-postası budur, adı şudur" diyen kriptolu bir token (Assertion) döner.

### 2.1 Moodle SAML2 Eklentisi Kurulumu

Moodle çekirdeğinde SAML2 yoktur, popüler olan `auth_saml2` (Catalyst IT) eklentisini Git ile kurmalıyız:

```bash
cd /var/www/moodle/auth
sudo git clone https://github.com/catalyst/moodle-auth_saml2.git saml2
sudo chown -R root:www-data saml2
sudo -u www-data php8.3 /var/www/moodle/admin/cli/upgrade.php --non-interactive
```

---

Artık Moodle giriş sayfasında kullanıcı adı/şifre yerine "Microsoft ile Giriş Yap" şeklinde kurumsal bir buton olacaktır.

---

## 3. Moodle Üzerinden BBB'ye Otomatik SSO

Peki kullanıcılarınız Moodle'a Microsoft Single Sign-On ile giriş yaptıktan sonra, dersteki BBB etkinliği için tekrar Greenlight'a girmek zorunda mı?

**HAYIR.**

Moodle ile BBB arasındaki LTI tabanlı `mod_bigbluebuttonbn` eklentisi arka planda kendi güven tünelini (`Secret Key` ve `Checksum`) kullanır.
Öğrenci Moodle'da "Toplantıya Katıl" butonuna basar, Moodle API aracılığıyla BBB sunucusuna şu komutu gönderir:

> *"Benim sistemimde Ahmet Yılmaz (<ahmet@sirket.com>) adında, Viewer (Öğrenci) rolüne sahip ve doğrulanmış bir kullanıcım var. Bu kullanıcı senin internalID'si XYZ-123 olan odana girmek istiyor. Bu kullanıcıyı kabul et."*

Bu sayede, kullanıcılar Moodle'a başarılı şekilde kimlik doğruladıklarında (ister LDAP, ister SAML, ister yerel hesap olsun), BBB'ye bir daha şifre girmeden, otomatik olarak isimleri ve rolleriyle (Öğretmenler moderatör olarak) bağlanırlar.

> [!TIP]
> **Tavsiye Edilen Mimari:** Eğer ana eğitim platformunuz Moodle ise, öğrencilere Greenlight (BBB arayüzü) adresini hiç vermeyin. `mod_bigbluebuttonbn` köprüsünü kurun. Tüm öğrenciler sadece `moodle.sirket.com` adresine Microsoft (SAML) hesaplarıyla giriş yapsın ve oradan direkt odalara bağlansın. Bu, kimlik karmaşasını sıfıra indirir.

---

## 4. Kullanıcı Temizliği (Life-Cycle Management)

SSO ile içeri binlerce kişi aktardınız, peki kişi şirketten/üniversiteden ayrıldığında Moodle'dan nasıl düşecek?

**Moodle Auth Plugin Ayarları:**
`Auth plugins` altında bulunan `Remove external users` seçeneğini **Suspend internal** olarak ayarlayın ve Sistem Cron'unun bu eşitlemeyi gece 03:00'te yapmasını (`Sync users` taskı) sağlayın. Böylece öğrenci AD'den silindiği gece Moodle'dan da yetkileri alınmış olur (`suspended`), ancak Moodle'daki sınav notları ve forum yazıları raporlama amacıyla korunur. Asla "Delete" seçeneğini kullanmayın.
