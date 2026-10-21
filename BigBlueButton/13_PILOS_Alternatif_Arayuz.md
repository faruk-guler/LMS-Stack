# Bölüm 13: PILOS — Greenlight'a Alternatif Akademik Ön Yüz (Frontend)

BigBlueButton'ı doğrudan kullanıcılarla buluşturmak için resmi arayüz **Greenlight** v3'tür. Ancak, özellikle üniversiteler ve akademik ortamlar için Almanya'daki TH Mittelhessen Uygulamalı Bilimler Üniversitesi (THM) tarafından geliştirilmiş, güçlü bir açık kaynak alternatif bulunmaktadır: **PILOS** (Platform for Interactive Live-Online Seminars).

## 13.1 PILOS Nedir ve Neden Geliştirildi?

PILOS, Covid-19 pandemisi sırasında öğrencilerin, öğretmenlerin ve personelin deneyimlerine dayanarak, dijital sınıflar ve grup çalışmaları için modern ve esnek bir video konferans yönetim sistemi ihtiyacıyla ortaya çıkmıştır.

Orijinal Greenlight projesinin çözemediği veya o günün şartlarında (Greenlight v2 zamanında) eklenmesi zor olan spesifik akademik gereksinimleri karşılamak amacıyla "Greenlight'a benzer bir arayüz ama daha esnek" mottosuyla geliştirilmiştir.

## 13.2 Greenlight ile PILOS Karşılaştırması

İkisi de kullanıcıların sisteme giriş yapmasını, oda açmasını ve ders başlatmasını sağlar. Temel farklar şunlardır:

* **Kullanıcı Kitlesi:** Greenlight daha genel kitlelere ve şirketlere hitap ederken, PILOS doğrudan üniversiteler, okullar ve akademik sınıf/ders yönetimi ihtiyaçlarına odaklanır.
* **Geliştirici:** Greenlight, BBB'nin kendi çekirdek ekibi (Blindside Networks) tarafından; PILOS ise THM Üniversitesi tarafından geliştirilmektedir.
* **Özelleştirme:** PILOS, özellikle belirli bir üniversitenin iç dinamiklerine (LDAP şemaları, özel oda izinleri) uyarlanabilmesi için bazı farklı esneklikler sunar.

> [!NOTE]
> Moodle veya Canvas gibi LMS sistemlerini kullanmıyorsanız ve sadece bağımsız okullarınız için bir "Sanal Sınıf Portalı" kuruyorsanız, Greenlight (v3) yerine alternatif olarak PILOS'u da (v4.x) değerlendirebilirsiniz.

---

## 13.3 Sistem Gereksinimleri

| Gereksinim | Açıklama |
| :--- | :--- |
| Hostname (FQDN) | Tam nitelikli alan adı (ör: `pilos.sirket.com`) |
| SSL Sertifikası | Geçerli HTTPS sertifikası (Let's Encrypt veya kurumsal CA) — üretim/kurulum: Certs Rehberi (`Certs/Sertifika_Rehberi.md`) |
| Reverse Proxy | Nginx (önerilen) veya Apache |
| Docker | Docker Engine + Compose Plugin |
| BBB Sunucusu | Mevcut çalışan bir BigBlueButton (v3.x veya v4.0) kurulumu |

---

## 13.4 Kurulum (Docker Compose ile)

PILOS, Docker imajı olarak dağıtılır. Varsayılan ayarlarla kurulur, sonra arayüz veya config dosyaları üzerinden özelleştirilir.

> [!IMPORTANT]
> PILOS, **Semantic Versioning** kullanır. Production ortamında her zaman sabit bir majör sürüm etiketi kullanın (ör: `v4`). `latest` etiketini production'da kullanmayın — majör sürümler arası otomatik yükseltme desteklenmez.

### 13.4.1 Proje Dizinini Oluşturma

```bash
mkdir pilos && cd pilos
```

### 13.4.2 Docker İmajını Tanımlama ve Dosyaları Çıkarma

```bash
# Kullanılacak sürümü belirleyin (ör: v4)
export IMAGE="pilos/pilos:v4"

# docker-compose.yml ve .env dosyalarını Docker imajından kopyalayın
docker run --rm $IMAGE cat ./.env.example > .env
docker run --rm $IMAGE cat ./docker-compose.yml > docker-compose.yml
sed -i "s|CONTAINER_IMAGE=.*|CONTAINER_IMAGE=$IMAGE|g" .env
```

### 13.4.3 Temel Yapılandırma (.env Dosyası)

#### APP_KEY (Zorunlu — Şifreleme Anahtarı)

Oturumları, çerezleri ve URL'leri şifrelemek için güvenli bir anahtar oluşturun:

```bash
docker run --rm --entrypoint pilos-cli $IMAGE key:generate --show
```

Çıktıyı `.env` dosyasındaki `APP_KEY` satırına yapıştırın:

```env
APP_KEY=base64:WJKM8YsGNutfcc3+2q/xiWE9Sus8GcbNVvTKFgdYgPw=
```

#### APP_URL (Zorunlu — Alan Adı)

```env
APP_URL=https://pilos.sirket.com
```

#### Veritabanı Şifresi

```bash
# Güvenli bir rastgele şifre oluşturun
openssl rand -hex 24
```

Çıktıyı `.env` dosyasındaki `DB_PASSWORD` satırına yapıştırın.

### 13.4.4 PostgreSQL Kullanımı (Önerilen)

PILOS varsayılan olarak MariaDB ile gelir. PostgreSQL kullanmak isterseniz `docker-compose.yml` dosyasındaki `db` servisini şu şekilde değiştirin:

```yaml
db:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    volumes:
        - ./db:/var/lib/postgresql
    environment:
        POSTGRES_USER: "${DB_USERNAME}"
        POSTGRES_PASSWORD: "${DB_PASSWORD}"
        POSTGRES_DB: "${DB_DATABASE}"
    healthcheck:
        test: ["CMD-SHELL", "pg_isready"]
        retries: 3
        timeout: 5s
```

`.env` dosyasında da veritabanı bağlantısını güncelleyin:

```env
DB_CONNECTION=pgsql
DB_HOST=db
DB_PORT=5432
```

---

## 13.5 Reverse Proxy Yapılandırması

PILOS kendi Nginx web sunucusunu içerir ancak konteynerin portunun doğrudan internete açılmaması önerilir. Önüne bir reverse proxy koymalısınız.

> [!WARNING]
> PILOS varsayılan olarak `127.0.0.1:5000` portunda dinler. Bu portu dışa açmayın; reverse proxy üzerinden yönlendirin.

### 13.5.1 Nginx (Önerilen)

```nginx
location / {
    proxy_pass          http://127.0.0.1:5000;
    proxy_set_header    Host              $host;
    proxy_set_header    X-Forwarded-Host  $host;
    proxy_set_header    X-Forwarded-Port  $server_port;
    proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto $scheme;
    proxy_http_version  1.1;
}
```

### 13.5.2 Nginx ile Rate Limiting (İsteğe Bağlı)

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=5r/s;

location / {
    limit_req zone=api burst=20 delay=10;
    limit_req_log_level warn;
    limit_req_status 429;

    proxy_pass          http://127.0.0.1:5000;
    proxy_set_header    Host              $host;
    proxy_set_header    X-Forwarded-Host  $host;
    proxy_set_header    X-Forwarded-Port  $server_port;
    proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto $scheme;
    proxy_http_version  1.1;
}
```

### 13.5.3 Trusted Proxies (Güvenilir Proxy Ayarı)

Reverse proxy aynı sunucuda çalışıyorsa (varsayılan durumda Docker bridge ağı üzerinden gelir):

```env
TRUSTED_PROXIES=*
```

> [!CAUTION]
> `TRUSTED_PROXIES=*` yalnızca reverse proxy ile uygulama aynı sunucudaysa güvenlidir. Farklı sunucudaysa, proxy IP'sini veya subnet'i belirtin (ör: `TRUSTED_PROXIES=10.0.0.0/8`). Aksi takdirde saldırganlar `X-Forwarded-For` başlığını sahteleyebilir.

---

## 13.6 Başlatma ve İlk Yapılandırma

### 13.6.1 Uygulamayı Başlatma

```bash
docker compose up -d

# Başlatma loglarını izlemek için:
docker compose logs -f
```

### 13.6.2 Superuser (Yönetici) Hesabı Oluşturma

```bash
docker compose exec app pilos-cli users:create:superuser
```

> [!NOTE]
> `.env` dosyasında değişiklik yaptıktan sonra konteyneri yeniden başlatmanız gerekir: `docker compose restart`

---

## 13.7 Güvenlik Önerileri

### 13.7.1 HSTS (HTTP Strict Transport Security)

Reverse proxy seviyesinde HSTS başlığını etkinleştirin:

**Nginx:**

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
```

### 13.7.2 Loglama

Varsayılan olarak tüm hatalar Docker loglarına (`stderr`) yazılır. Kalıcı dosya tabanlı loglama için:

```env
# .env dosyasında:
LOG_CHANNEL=daily
```

`docker-compose.yml` dosyasına log dizinini ekleyin:

```yaml
x-docker-pilos-common: &pilos-common
    env_file: .env
    volumes:
        - "./storage/logs:/var/www/html/storage/logs"
```

---

## 13.8 İleri Düzey Kimlik Doğrulama ve JSON Mapping

PILOS, LDAP veya SAML2 üzerinden gelen kullanıcıların rollerini ve niteliklerini (attributes) yönetmek için gelişmiş bir JSON tabanlı eşleştirme sistemi kullanır. Bu özellik, Greenlight v3'ün sahip olmadığı, doğrudan "Regex" (düzenli ifade) ile rol atama esnekliği sunar.

### 13.8.1 Öznitelik ve Rol Eşleştirme (`auth.json`)

PILOS kurulum dizininde veya yapılandırma panelinde tanımlanan bu JSON yapısı, dış kaynaklı kullanıcı bilgilerini PILOS içindeki rollere bağlar:

```json
{
    "attributes": {
        "external_id": "cn",
        "first_name": "givenname",
        "last_name": "sn",
        "email": "mail",
        "groups": "memberof"
    },
    "roles": [
        {
            "name": "user",
            "disabled": false,
            "rules": [
                {
                    "attribute": "external_id",
                    "regex": "/^.*/i"
                }
            ]
        },
        {
            "name": "superuser",
            "disabled": false,
            "all": true,
            "rules": [
                {
                    "attribute": "email",
                    "regex": "/@its\\.university\\.org$/i"
                },
                {
                    "attribute": "groups",
                    "regex": "/^cn=admin,ou=Groups,dc=university,dc=org$/im"
                }
            ]
        }
    ]
}
```

### 13.8.2 Bu Yapı Ne Sağlar?

1. **Regex ile Dinamik Rol Atama:** Örneğin, sadece `@its.university.org` uzantılı e-postası olan kişileri veya LDAP'ta `admin` grubunda olanları otomatik olarak `superuser` (moderatör yetkili yönetici) yapabilirsiniz.
2. **Grup Bazlı Yetkilendirme:** LDAP üzerindeki `memberof` niteliğini okuyarak, üniversite hiyerarşisini doğrudan PILOS rollerine yansıtabilirsiniz.
3. **Esnek Eşleştirme:** LDAP'taki `cn` niteliğini PILOS'un `external_id` alanına bağlayarak benzersiz kimlik yönetimini garantilersiniz.

---

## 13.9 BBB Sunucusu Bağlantısı

PILOS kurulumu tamamlandıktan sonra yönetici panelinden (Superuser ile giriş yaparak) BBB sunucunuzu eklemeniz gerekir:

1. PILOS web arayüzüne `https://pilos.sirket.com` adresinden giriş yapın.
2. **Yönetim Paneli → Sunucu Ayarları** bölümüne gidin.
3. BBB API URL'sini (`https://bbb.sirket.com/bigbluebutton/`) ve Secret Key'i (`bbb-conf --secret` çıktısı) girin.

> [!TIP]
> PILOS birden fazla BBB sunucusu eklemeyi destekler — bu sayede Scalelite'e ihtiyaç duymadan kendi iç yük dengelemesini yapabilirsiniz.

---

## 13.10 Greenlight vs PILOS — Hangisini Seçmeli?

| Kriter | Greenlight v3 | PILOS v4.x |
| :--- | :--- | :--- |
| Geliştirici | BBB çekirdek ekibi (Blindside) | THM Üniversitesi |
| Hedef kitle | Genel kurumsal / eğitim | Üniversiteler / akademik |
| Veritabanı | PostgreSQL | MariaDB veya PostgreSQL |
| Kimlik doğrulama | OIDC (Keycloak gerekli) | LDAP, SAML2, OIDC (doğrudan) |
| Çoklu BBB sunucu | Yok (Scalelite gerekir) | Dahili LB |
| Eklenti sistemi | Yok | Eklenti desteği var |
| Lisans | LGPL-3.0 | LGPL-2.1 |

Eğer kurumunuzun "resmi destekli", stabil bir kurumsal ürüne ihtiyacı varsa **Greenlight v3**'e (Bölüm 12); özel akademik ayarlara, doğrudan LDAP desteğine ve daha butik bir deneyime ihtiyacı varsa **PILOS**'a yönelebilirsiniz.

---

> [!TIP]
> **Resmi Kaynaklar:**
>
> * [PILOS Dokümantasyonu](https://thm-health.github.io/PILOS/docs/administration/intro)
> * [PILOS GitHub](https://github.com/THM-Health/PILOS)
> * [PILOS Güncelleme Rehberi](https://thm-health.github.io/PILOS/docs/administration/upgrade)
