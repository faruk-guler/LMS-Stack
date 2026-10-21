# Güvenlik Rehberi — moodle-bbb-stack (Sertleştirme Temeli · Defense-in-Depth)

Bu dosya yığındaki **platform-geneli güvenlik sertleştirme (hardening) temelinin tek kaynağıdır**: işletim sistemi, SSH, güvenlik duvarı (UFW), Fail2Ban, kernel/sysctl, dosya izin modeli, sırlar (secret) yönetimi, HTTP güvenlik başlıkları, güncelleme/yama disiplini ve WAF/CDN uyarıları — Moodle + BigBlueButton + Scalelite + Redis + veritabanları için ortak.

> **Örtüşmeyi önleyen sınır (tek kaynak):**
> - **TLS/sertifika üretimi & kurulumu** → `Certs/Sertifika_Rehberi.md` (burada yalnızca TLS *politika* başlıkları var).
> - **Kimlik/SSO (LDAP · SAML2 · OIDC/Keycloak)** → `Integrations/01_Kimlik_Hizmetleri_ve_SSO.md`.
> - **Redis güvenliği** (`requirepass`/`bind`/TLS) → `Redis/Redis_Rehberi.md` §2.
> - **Ürüne özgü ayarlar** (Moodle `config.php`, BBB `bbb-conf`/port matrisi) ilgili bölümlerde kalır; ortak temel buradadır.

## İçindekiler
1. [Tehdit modeli ve defense-in-depth](#1-tehdit-modeli-ve-defense-in-depth)
2. [İşletim Sistemi sertleştirmesi](#2-işletim-sistemi-sertleştirmesi-ubuntu)
3. [SSH sertleştirmesi](#3-ssh-sertleştirmesi)
4. [Güvenlik duvarı — UFW](#4-güvenlik-duvarı--ufw)
5. [Fail2Ban (brute-force koruması)](#5-fail2ban-brute-force-koruması)
6. [Kernel / sysctl sertleştirmesi](#6-kernel--sysctl-sertleştirmesi)
7. [Dosya izin modeli ve sırlar](#7-dosya-izin-modeli-ve-sırlar)
8. [HTTP güvenlik başlıkları](#8-http-güvenlik-başlıkları)
9. [Güncelleme / yama disiplini](#9-güncelleme--yama-disiplini)
10. [WAF / CDN ve WebRTC tuzakları](#10-waf--cdn-ve-webrtc-tuzakları)
11. [Kontrol listesi](#11-kontrol-listesi)

> Not: Her konu için **satır içi, kopyala-yapıştır** komut/örnekler aşağıdaki ilgili bölümlerde verilmiştir (ayrı örnek config dosyası yoktur).

---

## 1. Tehdit modeli ve defense-in-depth

Kurumsal LMS + canlı sınıf yığını **öğrenci/personel kişisel verisi** (KVKK/GDPR) ve **canlı yayın altyapısı** barındırır; saldırgan için caziptir. Katmanlı savunma (defense-in-depth):

```
Edge (DNS/DNSSEC, WAF/CDN, TLS) → Network (UFW/VIP, port azaltma) → Host (OS/SSH/sysctl/apparmor) →
Platform (fail2ban, least-privilege) → App (güncel Moodle/BBB, eklenti tedariki) → Data (şifre, yedek, sırlar) → İzleme/algılama
```

**Temel ilkeler:** en az yetki (least privilege), saldırı yüzeyini küçültme (kullanılmayan port/servis kapalı), varsayılan-güvenli (deny by default), güncel tutma, gözlemlenebilirlik (log+alarm → `Integrations/03`, `Monitoring`).

---

## 2. İşletim Sistemi sertleştirmesi (Ubuntu)

```bash
# Otomatik güvenlik yamaları
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# Saat doğruluğu (log/oturum token tutarlılığı için) — bkz. Moodle/10
sudo timedatectl set-ntp true

# Ek denetim (auditd) ve MAC (AppArmor — Ubuntu'da gelir)
sudo apt install -y auditd apparmor-profiles apparmor-utils
sudo systemctl enable --now auditd
```

- **Yalnızca gerekli paket/port**: kurulumda `-g` Greenlight, DB vb. — kullanılmayan servisi kapatın.
- **Ayrı kullanıcı + sudo**: günlük işler için `root` yerine yetkili kullanıcı; SSH'ta root girişi kapalı (§3).
- **Docker (BBB 4.0)**: rootless mümkün değilse konteyner ağlarını UFW ile kısıtlayın; imajları sabit sürüm etiketiyle çekin (latest'ten kaçın).

---

## 3. SSH sertleştirmesi

`/etc/ssh/sshd_config.d/hardening.conf`:

```sshd_config
PermitRootLogin no
PasswordAuthentication no          # yalnızca anahtar
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
AllowTcpForwarding no              # gerekiyorsa açın (tunnel)
X11Forwarding no
PermitEmptyPasswords no
# Erişimi kurumsal IP/grupla sınırlandırmak için (opsiyonel):
# AllowUsers deploy@10.0.0.0/24
```

```bash
sudo ssh-keygen -t ed25519 -C "admin@kurum"     # anahtarla giriş
sudo systemctl reload ssh
```

> [!WARNING]
> Anahtar tabanlı girişi **etkinleştirmeden** `PasswordAuthentication no` yapmayın; kendinizi kilitleyebilirsiniz. Önce test oturumunu açık tutun.

---

## 4. Güvenlik duvarı — UFW

İlke: **gelen varsayılan-deny, yalnızca gereken portlar açık**, SSH mümkünse rate-limit'li veya kurumsal IP'ye sabitli. Port matrisi ürün bazında değişir (Moodle 80/443; BBB 80/443 + UDP medya aralığı + 3478/5349 TURN; DB/Redis **dışa kapalı**).

Önemli kural: **PostgreSQL (5432) ve Redis (6379) asla 0.0.0.0'a açılmaz** — yalnız uygulama node'larına `allow from <subnet>`.

```bash
# Temel posture (rolüne göre gereksiz portları ekleyip çıkarın)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH           # daha sıkı: sudo ufw allow from <kurumsal-net> to any port 22 proto tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable && sudo ufw status verbose
```

```bash
# Örnek: DB/Redis'i iç ağa sabitle
sudo ufw allow from 10.0.0.0/24 to any port 5432 proto tcp
sudo ufw allow from 10.0.0.0/24 to any port 6379 proto tcp
```

> [!IMPORTANT]
> **SIP/UDP 5060 ve 3478** gibi VoIP/TURN portlarını internete açmak zorundaysanız yalnızca SIP/TURN sağlayıcınızın IP aralığına kısıtlayın (aksi halde "toll fraud" / relay istismarı) — bkz. `BigBlueButton/03`.

---

## 5. Fail2Ban (brute-force koruması)

SSH ve web oturum tekrarına karşı. Kurulum ve örnek `/etc/fail2ban/jail.local`:

```bash
sudo apt install -y fail2ban
sudo nano /etc/fail2ban/jail.local
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 10.0.0.0/24   # kurumsal çıkış/ağ IP'lerini ekleyin

[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s

[nginx-limit-rps]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/error.log
maxretry = 5
```

> [!NOTE]
> NAT/ortak-IP arkasındaki öğrenci havuzlarını kilitlememek için `bantime`/`maxretry`'yi makul tutun ve `ignoreip`'e kurumsal çıkış IP'lerini ekleyin. Moodle/BBB özgü filtreler için `filter.d/` altındaki `nginx-botsearch` vb. satırları log formatınıza göre uyarlayın.

---

## 6. Kernel / sysctl sertleştirmesi

Kalıcı ayar `/etc/sysctl.d/99-hardening.conf` içinde tutulur ve `sudo sysctl --system` ile uygulanır:

```ini
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
fs.protected_symlinks = 1
fs.protected_hardlinks = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
```

Kapsadığı başlıklar: `kernel.kptr_restrict=2`, `kernel.dmesg_restrict=1`, `fs.protected_symlinks/hardlinks/def_links=1` (yarış/coğrafya saldırılarına karşı), `net.ipv4.tcp_syncookies=1`, `rp_filter=1`, yönlendirme (IP redirect/router) kapalı, `accept_source_route=0`. Ayrıca çekirdek parametreleri **AppArmor/SELinux** ile birlikte anlam taşır; Ubuntu'da AppArmor default enforce profilleriyle gelir.

---

## 7. Dosya izin modeli ve sırlar

**Yığın genelinde ortak model:** kod **salt-okunur + root sahipliği**, web kullanıcısı (`www-data`) **yazamaz**; yazılabilir tek yer veri dizinleri (`moodledata`, BBB `/var/bigbluebutton`). Detay ürünlerde: `Moodle/06_Guvenlik_ve_Hardening.md`, `Moodle/04_Dosya_Yonetimi_ve_Moodledata.md`.

```bash
# Örnek genel model (Moodle kodu)
sudo chown -R root:www-data /var/www/moodle
sudo find /var/www/moodle -type d -exec chmod 750 {} \;
sudo find /var/www/moodle -type f -exec chmod 640 {} \;
```

**Sırlar (secrets):**
- `.env` (Greenlight/Scalelite/PILOS), `config.php`, `bbb-conf --secret`, DB/Redis şifreleri **Git'e girmez** (`.gitignore`); izinler `chmod 600`.
- Mümkünse merkezi sır yönetimi (Vault / SOPS / cloud secret manager) ve **rotasyon**: BBB secret değişikliği `sudo bbb-conf --setsecret`, Moodle `passwordsalt`, Redis `requirepass`, Scalelite `SECRET_KEY_BASE`/`LOADBALANCER_SECRET`.
- Örnek/dummy anahtarları (bkz. `Certs/`) asla üretimde kullanmayın.

---

## 8. HTTP güvenlik başlıkları

Nginx katmanında ortak başlıklar (Moodle/BBB/Scalelite vhost'ları):

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;   # HSTS (önce iyi niyetli testle)
```

> [!WARNING] — BBB/medya tuzağı
> **`Permissions-Policy`** ile `camera=()` / `microphone=()` koymayın — BBB'nin kamera/mikrofon erişimini bloklar. Sadece `geolocation=()`, `payment=()` gibi kullanılmayanları kapatın. **CSP** uygularken BBB/Greenlight WebSocket ve kaynak izinlerini (`connect-src wss:`, `media-src`) bozmayacak şekilde kademeli devreye alın.

---

## 9. Güncelleme / yama disiplini

- **OS**: `unattended-upgrades` (§2) + periyodik `apt update && apt dist-upgrade`.
- **BBB**: minör yamalar için `bbb-install.sh`'i tekrar çalıştır (bkz. `BigBlueButton/10`); majörlerde temiz kurulum + kayıt migrasyonu (3.x→4.0 yerinde desteklenmez).
- **Moodle**: Git dalı ile sürüm yükseltme + DB upgrade (`Moodle/07`).
- **Bağımlılık/Eklenti tedariki**: yalnızca güvenilir kaynaktan (Moodle plugins, resmi BBB), **2+ yıldır güncellenmeyen** eklentiden kaçın (`Moodle/09`).
- **Rollback planı**: her yama öncesi snapshot + yedek (`Backups`).

---

## 10. WAF / CDN ve WebRTC tuzakları

- **Cloudflare/WAF "orange cloud" (proxied)** WebRTC/UDP medyayı ve 7443 WebSocket'i kırar (kamera donar). BBB/Scalelite uçlarını **DNS-only (grey cloud)** bırakın; proxy şart ise **Spectrum/Enterprise** (UDP/TURN) gerekir.
- WAF kuralları **WebSocket ve RTP paketlerini düşürmeyecek** şekilde ayarlanmalı; OCR/DDoS eşikleri toplantı anında yanıltıcı olmamalı.
- Yönetim paneli (`/admin`, Greenlight, Scalelite) için **IP allowlist** (Nginx `allow/deny`) ve/veya VPN — `Moodle/06` ve `BigBlueButton/04`.

---

## 11. Kontrol Listesi

- [ ] `unattended-upgrades` açık; OS/BBB/Moodle güncel; majör geçiş planı var
- [ ] SSH: `PermitRootLogin no` + `PasswordAuthentication no` (anahtar doğrulandı)
- [ ] UFW: default deny; 5432/6379 **dışa kapalı**, iç ağa kısıtlı; SIP/TURN sağlayıcı-IP'li
- [ ] Fail2Ban: sshd + nginx jail'ları aktif; NAT-arkası için makul ban eşikleri
- [ ] sysctl: `/etc/sysctl.d/99-hardening.conf` uygulandı (§6); AppArmor profilleri enforce
- [ ] Dosya izinleri: kod root-owned/salt-okunur, `config.php`/`.env` `600`, sırlar Git'te YOK
- [ ] Sır rotasyonu tanımlı (bbb secret, salts, DB/Redis şifresi)
- [ ] HTTP başlıkları: HSTS/nosniff/XFO; `Permissions-Policy` kamera/mikrofonu KAPATMIYOR
- [ ] WAF/CDN: BBB/Scalelite grey-cloud; yönetim panelleri IP/VPN kısıtlı
- [ ] İzleme/algılama: log + alert (bkz. `Integrations/03`, `Monitoring`), yıllık sızma testi/VA taraması

---

### Çapraz referanslar (düz metin — yerel link yok)
- TLS/sertifika: `Certs/Sertifika_Rehberi.md`
- Kimlik/SSO: `Integrations/01_Kimlik_Hizmetleri_ve_SSO.md`
- Redis güvenliği: `Redis/Redis_Rehberi.md` (§2)
- Moodle dosya/izin & `config.php`: `Moodle/06_Guvenlik_ve_Hardening.md`, `Moodle/04_Dosya_Yonetimi_ve_Moodledata.md`
- BBB ağ/SIP/TURN portları: `BigBlueButton/03_Ag_Guvenlik_Duvari_ve_TURN_STUN.md`
- Güncelleme/yama: `BigBlueButton/10_Guncelleme_Yedekleme_ve_Guvenlik.md`, `Moodle/07_Yedekleme_ve_Surum_Yukseltme.md`
- HA (split-brain/VIP güvenliği): `Integrations/05_High_Availability_and_Failover.md`
