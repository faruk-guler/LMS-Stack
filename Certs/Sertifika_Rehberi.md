# Certs — Sertifika (SSL/TLS) Kurulum Rehberi

Bu dosya **moodle-bbb-stack** boyunca ihtiyaç duyulan **tüm sertifika işlemlerinin tek kaynağıdır** (single source of truth): CSR üretimi, **Let's Encrypt**, **ticari/küresel CA (Sectigo, GlobalSign, DigiCert vb.)**, **self-signed (test)** sertifikaların üretilmesi, Nginx'e (BBB / Moodle / Scalelite / Coturn) kurulması, doğrulanması ve yenilenmesi.

> Diğer bölümler (BigBlueButton 02, Moodle 02, Integrations 04) sertifika adımlarını burada anlatılan bu rehberin uygulanması olarak ele alır; ayrıntı yinelenmez.

## İçindekiler
- [1. Sertifika Kaynakları ve Hangisini Seçmeli](#1-sertifika-kaynakları-ve-hangisini-seçmeli)
- [2. Hazırlık: Dizin, SAN ve csr.conf](#2-hazırlık-dizin-san-ve-csrconf)
- [3. Özel Anahtar ve CSR Üretimi](#3-özel-anahtar-ve-csr-üretimi)
- [4. Yol A — Let's Encrypt (Otomatik)](#4-yol-a--lets-encrypt-otomatik)
- [5. Yol B — Ticari/Küresel CA (Sectigo vb.)](#5-yol-b--ticariküresel-ca-sectigo-vb)
- [6. Yol C — Self-Signed (yalnızca test)](#6-yol-c--self-signed-yalnızca-test)
- [7. Nginx'e Kurulum (BBB · Moodle · Scalelite · Coturn)](#7-nginxe-kurulum-bbb--moodle--scalelite--coturn)
- [8. Doğrulama](#8-doğrulama)
- [9. Yenileme (Renewal) ve Otomasyon](#9-yenileme-renewal-ve-otomasyon)
- [10. Kontrol Listesi](#10-kontrol-listesi)

---

## 1. Sertifika Kaynakları ve Hangisini Seçmeli

| Kaynak | Ne zaman | Ücret/özellik |
| :--- | :--- | :--- |
| **Let's Encrypt** | İnternete açık, otomatik, kısa ömürlü (90 gün) cert yeterliyse | Ücretsiz; `certbot` ile otomatik yenileme |
| **Ticari/Küresel CA (Sectigo vb.)** | Kurumsal güvenilirlik, wildcard, uzun süre, denetimli CA zinciri gerekiyorsa | Ücretli; **ara (intermediate) zinciri elle doğru kurmak şart** |
| **Self-Signed** | Sadece iç test/lab | Ücretsiz; **tarayıcı "güvensiz" der, WebRTC izinlerini bloklar** |

> [!IMPORTANT]
> Üretimde **self-signed KULLANMAYIN**. Tarayıcılar geçersiz TLS'de kamera/mikrofon izni vermez → BBB'de 1007/1020 benzeri medya hataları. Self-signed yalnız §6'daki test senaryosu içindir.

---

## 2. Hazırlık: Dizin, SAN ve csr.conf

Tüm yollar için ortak sertifika dizini:

```bash
sudo mkdir -p /etc/nginx/ssl
sudo chmod 700 /etc/nginx/ssl
```

**SAN (Subject Alternative Name) zorunludur.** Modern tarayıcılar CN'i değil **SAN**'ı esas alır. Her host için doğru DNS adını SAN'a yazın:

- BBB → `bbb.sirket.com`
- Moodle → `moodle.sirket.com`
- Scalelite → `myscalelite.sirket.com`
- Coturn → `turn.sirket.com` (ayrı sertifika)

Bu depo içindeki **`Certs/csr.conf`** bir CSR/self-signed şablonudur; `CN` ve `[alt_names]` (DNS.1, DNS.2, IP.1) bölümlerini kendi adlarınızla düzenleyin:

```ini
[req_distinguished_name]
CN = bbb.sirket.com          # gerçek sunucu adınız

[alt_names]
DNS.1 = bbb.sirket.com
DNS.2 = *.sirket.com         # isteğe bağlı wildcard
```

> DH param (istedeğe bağlı, güçlü güvenliğe katkı):
> ```bash
> sudo openssl dhparam -out /etc/nginx/ssl/dhp-2048.pem 2048
> ```

---

## 3. Özel Anahtar ve CSR Üretimi

Ticari CA (Yol B) için gereken **CSR**'ı ve özel anahtarınızı üretin. Anahtar sunucuda kalır, **CSR** metnini CA'ya gönderirsiniz:

```bash
# rsa:2048 hızlıdır; uzun vadeli/üst güvenlik için rsa:4096 veya -newkey ec:...(ECDSA)
openssl req -new -newkey rsa:2048 -nodes \
  -keyout /etc/nginx/ssl/bbb.key \
  -out    /etc/nginx/ssl/bbb.csr \
  -config Certs/csr.conf

sudo chmod 600 /etc/nginx/ssl/bbb.key        # özel anahtar KESİNlikle 600
openssl req -in /etc/nginx/ssl/bbb.csr -noout -text   # CSR içeriğini (SAN dahil) doğrula
```

> [!WARNING]
> `.key` (özel anahtar) dosyasını **asla Git deposuna commit etmeyin** ve CA'ya göndermeyin; yalnızca `.csr` gönderilir.

---

## 4. Yol A — Let's Encrypt (Otomatik)

**BBB'de en kolay yol:** `bbb-install.sh` zaten certbot ile Let's Encrypt sertifikasını otomatik alır (kurulumda `-e e-posta` vermeniz yeterli — bkz. BigBlueButton 02 (`BigBlueButton/02_Sistem_Gereksinimleri_ve_Kurulum.md`)). Elle/yeniden kurulum:

```bash
sudo apt install -y certbot python3-certbot-nginx
# BBB nginx vhost'unuzu kullanarak cert al:
sudo certbot --nginx -d bbb.sirket.com
# Wildcard isterseniz DNS-01:
sudo certbot certonly --manual --preferred-challenges dns \
  --agree-tos -m admin@sirket.com -d bbb.sirket.com -d '*.sirket.com'
```

Dosyalar `/etc/letsencrypt/live/<alanad>/` altında üretilir:
- `fullchain.pem` → `ssl_certificate`
- `privkey.pem` → `ssl_certificate_key`

Zamanlanmış yenileme (§9): `certbot renew` systemd timer'ı ile otomatik.

---

## 5. Yol B — Ticari/Küresel CA (Sectigo vb.)

Sectigo gibi **küresel bir CA'dan** sertifika aldığınızda akış: **CSR üret (§3) → CA'ya gönder → sertifika + ara (intermediate) paketini al → fullchain kur → doğrula.**

**5.1) CSR'ı gönder:** §3'te üretilen `/etc/nginx/ssl/bbb.csr` içeriğini Sectigo paneline yapıştırın; domain doğrulamasını (e-posta/DNS/CNAME) tamamlayın. CA size şunları verir:
- **Sunucu sertifikası**: `bbb.crt` (veya `.cer`)
- **Ara/CA bundle**: genelde 2 ara sertifika (ör. `SectigoRSADomainValidationCA.crt`, `UserTrust...crt`). **Kök (root) CA ayrıdır ve fullchain'e konmaz.**

**5.2) Fullchain'i DOĞRU sırayla birleştir:** önce sunucu cert'i, sonra aralar (root EKLEME):

```bash
# Ara paketin içeriğini SIRA ile düzenle: sunucu cert → ara-1 → ara-2
cat /etc/nginx/ssl/bbb.crt Sectigo_CA-bundle.crt | sudo tee /etc/nginx/ssl/fullchain.pem >/dev/null
```

> [!CAUTION]
> **En sık ticari-CA hatası:** intermediate zincirini eklemeyi unutmak. Zincir eksikse tarayıcı cert'i CA köküne bağlayamaz → BBB/Moodle'da **"güvensiz" + kamera/mikrofon izni engeli** ("Chain issues: Incomplete"). Sectigo bundle'ını **her zaman ekleyin**, root'u eklemeyin.

**5.3) Anahtarı koru ve §7'ye geç** (`fullchain.pem` + `bbb.key`).

---

## 6. Yol C — Self-Signed (yalnızca test)

**Sadece iç lab/test.** `csr.conf` kullanarak 365 günlük kendi-kendini-imzalayan cert üretir (CSR/CA turu yok):

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/bbb.key \
  -out    /etc/nginx/ssl/bbb.crt \
  -config Certs/csr.conf -extensions v3_req
sudo chmod 600 /etc/nginx/ssl/bbb.key
```

İstemcilerin bu cert'i "güvenilir kök CA" olarak tanıması (Linux):

```bash
sudo cp /etc/nginx/ssl/bbb.crt /usr/local/share/ca-certificates/bbb-sirket.crt
sudo update-ca-certificates
```

> [!WARNING]
> Self-signed üretimde de §2'deki SAN'lar doğru olmalı (CN + subjectAltName). Yoksa modern tarayıcı `NET_ERR_CERT_*` ile yine reddeder.

---

## 7. Nginx'e Kurulum (BBB · Moodle · Scalelite · Coturn)

Hangi yolu kullanırsanız kullanın, Nginx vhost'unda **`ssl_certificate` = fullchain** (veya self-signed'de tek `.crt`), **`ssl_certificate_key` = `.key`** olmalı. Ortaya güvenli bir TLS blok örneği:

```nginx
ssl_certificate     /etc/nginx/ssl/fullchain.pem;   # Yol B/A: fullchain — Yol C: bbb.crt
ssl_certificate_key /etc/nginx/ssl/bbb.key;
# ssl_dhparam       /etc/nginx/ssl/dhp-2048.pem;   # §2'de ürettiyseniz açın
ssl_protocols       TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;
ssl_session_cache   shared:SSL:10m;
ssl_session_timeout 10m;
# Mozilla "Intermediate" önerisi — TLS1.3 + güvenli TLS1.2 AES-GCM/ChaCha20 şifre paketleri:
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;  # HSTS — doğrulandıktan sonra açın
```

**Hedef bazında uygulama:**

- **BigBlueButton** → vhost: `/etc/nginx/sites-available/bigbluebutton`. Değişiklik sonrası:
  ```bash
  sudo nginx -t && sudo systemctl reload nginx && sudo bbb-conf --restart
  ```
  (İpucu: `bbb-conf --restart` şart; eski cert cache'i ve medya/SIP TLS bağlantıları tazelemeli — bkz. BBB 16 §16.5 (`BigBlueButton/16_Sik_Karsilasilan_Hatalar_ve_Kriz_Yonetimi.md`).)

- **Moodle** → site vhost'unda aynı bloğu kullanın (mevcut config: Moodle 02 (`Moodle/02_Sunucu_Kurulumu_ve_Gereksinimler.md`)); `wwwroot` zaten `https://` olmalı.
  ```bash
  sudo nginx -t && sudo systemctl reload nginx
  ```

- **Scalelite (reverse proxy)** → host Nginx'te (bkz. Integrations 04 §5 (`Integrations/04_Scalelite_Balancer.md`)) `ssl_certificate`/`ssl_certificate_key` aynı fullchain/key'e işaret eder.

- **Coturn (TURN/STUN)** → `/etc/turnserver.conf` içinde ayrı cert verin (`cert=/etc/turn/ssl/turn.crt`, `pkey=...`), `listening-port` 443/TLS veya 5349; `turn.sirket.com` SAN'lı ayrı cert (bkz. BBB 03 (`BigBlueButton/03_Ag_Guvenlik_Duvari_ve_TURN_STUN.md`)).

> [!IMPORTANT]
> Her servis ayrı host/DNS ise **her biri için ayrı (veya wildcard) sertifika** gerekir. Tek `bbb.crt`'yi `moodle.sirket.com`'a koymayın — hostname mismatch hatası alırsınız.

---

## 8. Doğrulama

```bash
# 8.1) Yerel: private key ↔ cert eşleşmesi (MD5 karşılaştır)
openssl x509 -noout -modulus -in /etc/nginx/ssl/fullchain.pem | openssl md5
openssl rsa  -noout -modulus -in /etc/nginx/ssl/bbb.key        | openssl md5

# 8.2) Zincir bütünlüğü (ticari CA) — eksik intermediate'i yakalar
openssl verify -CAfile <(cat Sectigo_CA-bundle.crt) /etc/nginx/ssl/bbb.crt

# 8.3) Canlı: sunucu doğru zinciri sunuyor mu? (fullchain'de aralar var mı)
openssl s_client -connect bbb.sirket.com:443 -servername bbb.sirket.com -showcerts </dev/null 2>/dev/null \
  | grep -E "depth=|verify|s:subject|Verify return code"
# Beklenen: "Verify return code: 0 (ok)" ve depth'te root'a kadar inmeli

# 8.4) TLS 1.2/1.3 çalışıyor mu
openssl s_client -connect bbb.sirket.com:443 -tls1_2 </dev/null 2>&1 | head -n 5
```

> [!TIP]
> En kesin test: [SSL Labs Server Test](https://www.ssllabs.com/ssltest/) → "A" notu **ve** "Chain issues: None" hedefi. "Incomplete chain" görürsen §5.2'deki `fullchain.pem`'de ara sertifika eksik demektir.

---

## 9. Yenileme (Renewal) ve Otomasyon

- **Let's Encrypt:** certbot kuruluysa systemd timer otomatik yeniler; test:
  ```bash
  sudo certbot renew --dry-run
  sudo systemctl list-timers | grep certbot
  ```
- **Ticari CA:** süresi dolmadan yeni cert ile §5.2–§7'yi tekrarlayıp `fullchain.pem`'i değiştirin; `reload nginx` (BBB'de `bbb-conf --restart`).
- **Self-signed (test):** süresi dolunca §6'yı yeniden çalıştırın.
- **Gözetim:** süresi dolan/eksik zincir → Integrations 03 (`Integrations/03_Merkezi_Log_Yonetimi.md`) + Prometheus (bkz. BBB 09 (`BigBlueButton/09_Izleme_Loglama_ve_SorunGiderme.md`)) ile **expiring-cert + HAProxy/keepalived HA** senaryosu için Integrations 05 (`Integrations/05_High_Availability_and_Failover.md`).
- **Yüksek erişilebilirlik:** girişte VIP/HAProxy varsa cert yalnız **LB'de** (veya her düğümde aynı fullchain) durur; failover'da zincirin eksiksiz kopyalandığından emin olun.

---

## 10. Kontrol Listesi

- [ ] Her host için doğru **SAN** (DNS.1/DNS.2) üretildi — `openssl req -in bbb.csr -text` ile görünüyor
- [ ] `ssl_certificate` → **fullchain** (sunucu + intermediate'lar), **root yok**
- [ ] `ssl_certificate_key` → doğru `.key`; `chmod 600` ve Nginx okuyabiliyor (genelde root)
- [ ] Özel anahtar **Git'de yok**; `.crt/.key` gerçek üretim anahtarı commit edilmemiş
- [ ] `openssl verify` ve SSL Labs → **chain complete / return code 0**
- [ ] BBB'de `bbb-conf --restart` sonrası **kamera/mikrofon izni** sorunsuz (WebRTC TLS OK)
- [ ] Coturn için **ayrı** cert (turn.sirket.com) ve BBB'nin `-c <turn-host>:<secret>` yapılandırması güncel (`bbb-conf --setturn` diye bir bayrak yoktur)
- [ ] Yenileme otomasyonu (certbot timer veya takvimli CA yenileme) kuruldu
- [ ] HA var ise VIP/failover düğümlerinde aynı `fullchain.pem` bulunuyor

---

### Kaynak ve çapraz referanslar
- **`Certs/csr.conf`** — bu rehberin §3/§6'sında kullanılan SAN şablonu.
- Sertifika kurulumunun servis özelindeki yerleri: **BigBlueButton 02** (kurulum + `bbb-install -e`), **Moodle 02** (Nginx vhost), **Integrations 04 §5** (Scalelite reverse proxy), **BBB 03** (Coturn), **BBB 16 §16.5** (süre dolması / kriz yenileme).
