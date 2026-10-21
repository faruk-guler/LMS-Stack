# Bölüm 12: Greenlight v3 İleri Düzey Yönetim ve Kullanıcı Entegrasyonları (2026 Standartları)

Eğer LMS (Moodle, Canvas) platformlarına sahip değilseniz, 2026 yılı standartlarında işletmeniz veya okulunuz için en ideal arayüz BigBlueButton'ın resmi ön yüzü olan **Greenlight v3**'tür.

Greenlight v3, önceki sürümlerden farklı olarak daha modern (React frontend, Ruby backend, PostgreSQL veritabanı) ve daha yetenekli bir mimariye sahiptir.

## 12.1 Yönetici Paneli ve Rol Yapılandırması (RBAC)

Greenlight v3, Rol Tabanlı Erişim Kontrolü (Role-Based Access Control) sunar.

- **Super Administrator:** Sitenin ana rengini (branding), kayıt olma politikalarını ve tüm diğer kullanıcıları yönetir.
- **Administrator:** Oda oluşturabilir, diğerlerinin kayıtlarını (recordings) ve odalarını görebilir.
- **User:** Sadece kendi odasını yaratır, kaydını alır, toplantıya davet linki üretir. Kullanıcı diğer odaları göremez.
- **Guest (Misafir):** Hesabı olmayan ancak link ile ("Katıl") butonuna basarak gelen kişilerdir. Greenlight bu kişilere oturum açtırmaz, sadece toplantıya sokar.

`Site Settings` altından "Roller" kısmına gelip yeni roller yaratabilir (Örneğin: "Öğretmen" rolü, "Müdür" rolü) ve bunlara farklı oda açma limitleri koyabilirsiniz.

## 12.2 Kurumsal Kimlik Doğrulama: LDAP ve Active Directory (Keycloak OIDC)

Eğer 500 personelli bir şirketiniz varsa ve personelin Greenlight'a mevcut Active Directory (şirket) şifresiyle girmesini istiyorsanız bir entegrasyon şarttır.

> [!WARNING]
> **Önemli Değişiklik:** Greenlight v2'deki doğrudan LDAP entegrasyonu (`LDAP_SERVER=...`) Greenlight v3 ile birlikte **kaldırılmıştır**. v3 mimarisi, güvenliği ve esnekliği artırmak amacıyla yalnızca OpenID Connect (OIDC) ve OAuth2 (Google/O365) protokollerini destekler.

Bu nedenle LDAP sunucunuzu bağlamak için araya ücretsiz bir açık kaynak Kimlik Sağlayıcı (IdP) olan **Keycloak** kurmalısınız.

Detaylı mimari ve Keycloak kurulum adımları için Kimlik Hizmetleri ve SSO (`Integrations/01_Kimlik_Hizmetleri_ve_SSO.md`) dokümanını temel alın.

**Yeni Kimlik Doğrulama Mimarisi:**

```text
Active Directory / LDAP Sunucusu
        ↑ ↓ (LDAPS Bind)
    Keycloak (Kimlik Sağlayıcı / User Federation)
        ↑ ↓ (OpenID Connect - OIDC)
   Greenlight v3 (Sadece OIDC ile Konuşur)
```

**Adım Adım Entegrasyon Mantığı:**

### Adım 1: Keycloak Kurulumu (Docker Compose)

Şirket ağınızda ayrı bir sunucu (veya BBB sunucusu) üzerinde Keycloak çalıştırın:

```bash
sudo mkdir -p /opt/keycloak && cd /opt/keycloak
sudo nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  keycloak-db:
    image: postgres:16
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: guclu_keycloak_sifresi
    volumes:
      - keycloak_db:/var/lib/postgresql/data
    restart: unless-stopped

  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    command: start --optimized
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: guclu_keycloak_sifresi
      KC_HOSTNAME: keycloak.sirket.com
      KC_PROXY_HEADERS: edge
      KC_HTTP_ENABLED: "true"
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: Admin_Sifresi_123!
    ports:
      - "8080:8080"
    depends_on:
      - keycloak-db
    restart: unless-stopped

volumes:
  keycloak_db:
```

```bash
docker compose up -d
# Panel: http://keycloak.sirket.com:8080  (admin / Admin_Sifresi_123!)
```

> [!IMPORTANT]
> Üretim ortamında Keycloak'ı Nginx reverse proxy arkasında HTTPS ile çalıştırın. Greenlight ile OIDC iletişimi HTTPS olmadan çalışmaz.

Admin panelinde: sol üstteki **realm seçiciden → Create Realm** ile **"BigBlueButton"** adında yeni bir Realm oluşturun. (Dikkat: `Realm Settings` mevcut realm'i düzenler, yeni realm oluşturmaz.)

1. **Keycloak - LDAP Bağlantısı:** Admin panelinde `User Federation` → `Add LDAP provider`. Sunucu IP, Bind DN (`CN=ReadUser,OU=Admin,DC=sirket,DC=local`) ve parolayı girin. `Synchronize all users` ile AD kullanıcılarını Keycloak'a aktarın.
2. **Keycloak - Greenlight OIDC İstemcisi:** `Clients` → `Create client` → Client ID: `greenlight`, `Client authentication: ON`. Oluşan **Client Secret**'ı kopyalayın.
3. **Greenlight v3 (`.env`) Yapılandırması:** Greenlight sunucunuza gidip `.env` dosyasına OIDC parametrelerini ekleyin:

```env
# Varsayılan kimlik doğrulamasını OIDC yapın
AUTH_METHOD=openid_connect

# Keycloak (OIDC) Parametreleri
OPENID_CONNECT_CLIENT_ID=greenlight
OPENID_CONNECT_CLIENT_SECRET=keycloak_panelinden_aldiginiz_gizli_sifre
OPENID_CONNECT_ISSUER=https://keycloak.sirket.com/realms/BigBlueButton
OPENID_CONNECT_UID_FIELD=preferred_username
```

Değişiklikten sonra `docker compose down && docker compose up -d` yapın. Artık "Giriş Yap" düğmesine basan personel, şık bir Keycloak ekranına yönlendirilecek; oraya AD şifresini girdiğinde doğrulama yapılıp Greenlight'a geri dönecektir.

## 12.3 OAuth2 (Google / Office 365) ile E-posta Üzerinden Giriş

Modern okullar "Google Workspace" (GSuite) veya "Microsoft 365" üzerinden e-posta kullanmaktadır.

> [!WARNING]
> Greenlight v2'deki `OFFICE365_KEY`, `OFFICE365_SECRET` gibi doğrudan OAuth2 parametreleri **v3'te kaldırılmıştır**. Greenlight v3, tüm harici kimlik doğrulamayı **OpenID Connect (OIDC)** üzerinden yapar.

**Microsoft 365 / Google ile Giriş (Greenlight v3 Yöntemi):**

İki seçeneğiniz vardır:

1. **Keycloak Üzerinden (Önerilen):** Bölüm 12.2'deki Keycloak kurulumunuzu yaptıysanız, Keycloak yönetim panelinden "Identity Providers" sekmesine gidip Google veya Microsoft'u ekleyin. Greenlight tarafında ek bir yapılandırma gerekmez; Keycloak giriş ekranında otomatik olarak "Microsoft ile Giriş Yap" butonu belirir.

2. **Doğrudan OIDC ile:** Microsoft Entra ID (Azure AD) portalından bir "App Registration" yapıp OIDC uç noktalarını (Issuer URL, Client ID, Client Secret) doğrudan Greenlight `.env` dosyasındaki `OPENID_CONNECT_*` parametrelerine girin:

```env
OPENID_CONNECT_CLIENT_ID=azure_application_id
OPENID_CONNECT_CLIENT_SECRET=azure_client_secret
OPENID_CONNECT_ISSUER=https://login.microsoftonline.com/TENANT_ID/v2.0
```

## 12.4 Yük Dengeleme: Greenlight + Scalelite

Greenlight v3'ün **kendi içinde yük dengeleyicisi yoktur**; yalnızca tek bir BBB uç noktasına (`BIGBLUEBUTTON_ENDPOINT`/`BIGBLUEBUTTON_SECRET`) bağlanır. Bu uç nokta doğrudan bir BBB sunucusu **değil**, havuzun önündeki **Scalelite** olmalıdır.

- `.env` içinde `BIGBLUEBUTTON_ENDPOINT=https://scalelite.sirket.com/bigbluebutton/api` ve `BIGBLUEBUTTON_SECRET=<Scalelite LOADBALANCER_SECRET>` tanımlayın.
- Scalelite, yeni toplantıları otomatik olarak havuzdaki en boş worker'a (`bbb01`, `bbb02`, …) dağıtır; Scalelite kurulumu için Bölüm 8 ve `Integrations/04_Scalelite_Balancer.md`'e bakın.
- Arayüz düzeyinde **çoklu BBB sunucusunu kendi içinde** dengelemek isterseniz bunu destekleyen ürün Greenlight değil **PILOS**'tur (bkz. `BigBlueButton/13_PILOS_Alternatif_Arayuz.md`). Ortak NFS/S3 kayıt deposu her durumda ayrıca yapılandırılır.
