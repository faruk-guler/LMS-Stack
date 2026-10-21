# Bölüm 10: Güncelleme, Yedekleme ve Güvenlik (Sıkılaştırma)

BigBlueButton, aktif bir proje olduğundan sürekli güvenlik yamaları ve performans iyileştirmelerini barındıran minör veya majör güncellemeler alır.

## 10.1 BigBlueButton'ı Güncellemek (Upgrading)

BigBlueButton'ı güncellemenin iki farklı rotası vardır:

1. **Minör/Günlük Güncelleme (Örn: 4.0.0'dan 4.0.x'e veya 3.0.x yamaları):** Kurulum yaparken kullandığınız `bbb-install` komutunun aynısını tekrar çalıştırırsınız. Veya sadece `sudo apt update && sudo apt dist-upgrade` yazarsınız, ancak paket yolları değiştiyse `bbb-install` komutu daha garantilidir.
2. **Majör Güncelleme (Örn: 3.0'dan 4.0'a):** Büyük sürüm atlarken eski ayarlar (`/etc/bigbluebutton/...`) uyumsuzluk yaratabilir ve BBB 4.0 farklı bir işletim sistemi (Ubuntu 24.04) ile Docker tabanlı yeni bir medya yığını gerektirir. Bu sebeple **3.0 → 4.0 yerinde yükseltme desteklenmez**; temiz Ubuntu 24.04 kurup BBB 4.0'ı kurmak ve eski kayıtları "aktarmak" (Migrate) önerilen resmi yöntemdir. Eski sunucu formatlanmaz, yeni sunucu kurulur, DNS (bbb.sirket.com) yeni sunucuya kaydırılır.

> [!CAUTION]
> Herhangi bir güncelleme yapmadan önce (apt base güncelleme dahil), ortamda toplantı olmadığından emin olun ve sunucunun Snapshot'ını (Anlık Görüntü) alın!

## 10.2 BigBlueButton Yedeklemesi (Backup Stratejileri)

BBB'de yedeklenecek 3 hedef: **ayarlar** (`/etc/bigbluebutton`, `/etc/nginx`, `/var/www/bigbluebutton-default`), **Greenlight v3 Postgres DB + `.env`**, ve **kayıtlar** (`/var/bigbluebutton/published/` — TB'larca veri tutabilir; S3/NFS'e taşımak esastır).

> Otomatik betik, `rsync`/S3 hedefleri, retention, **PITR** ve **restore** adımları tek kaynakta: `Backups/Yedekleme_Rehberi.md` (§3, §4.2, §8.2).

## 10.3 Güvenlik Sıkılaştırması (Linux Hardening & WAF)

BigBlueButton 80, 443 ve geniş bir UDP aralığı kullanır. Klasik bir web sunucusundan daha agresif güvenlik önlemleri alırken (WAF vb) "Websocket / RTP paket düşmesin" diye ayar yapmalısınız.

> [!NOTE]
> **Ortak sertleştirme temeli** (UFW posture, SSH, Fail2Ban jail'ları, sysctl, WAF/CDN WebRTC tuzakları) için tek kaynak: `Security/Guvenlik_Rehberi.md`. Buradaki §10.3 yalnızca **BBB'ye özgü** hatırlatmaları (WAF grey-cloud, TURN relay istismarı) korur.

### 1. WAF / CDN (Cloudflare) — WebRTC tuzakları

**Cloudflare** veya benzeri bir WAF'ın arkasına BBB konulduğunda (Proxy mode "Orange cloud"):

- WebRTC (UDP paketleri) ve 7443 (WebSocket) bağlantıları takılır, kameralar donar, bağlanılamaz!
- BBB yalnızca **DNS-only ("Grey Cloud")** modunda çalıştırılmalıdır; proxy şart ise **Cloudflare Spectrum** (pahalı tier, UDP/TCP) gerekir.

### 2. TURN/Coturn Relay İstismarını Engelleme

Coturn (TURN) sunucusu başkalarınca gizli tünel olarak kullanılmasın diye yalnızca BBB sunucunuzun IP'sine/kimlik doğrulamasına izin verin (`turnserver.conf` içinde `allowed-peer-ip`) — bkz. `BigBlueButton/03`.

> **Genel sertleştirme temeli** (SSH, UFW posture + port kapatma, Fail2Ban jail'ları, sysctl, WAF/CDN ilkeleri) tek kaynakta: `Security/Guvenlik_Rehberi.md` (§4, §5, §10).

---
**İpucu:** Güvenlik önlemlerini uyguladıktan sonra bir sonraki bölüm olan **API Kullanımı ve Dış Entegrasyonlar** ile devam edin.
