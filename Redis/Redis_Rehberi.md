# Redis Rehberi — moodle-bbb-stack (Kurulum · Güvenlik · Performans · HA)

Bu dosya **Redis ile ilgili tüm işlemlerin tek kaynağıdır** (single source of truth): kurulum, güvenlik/sertleştirme, bellek/eviction politikası, kalıcılık (persistence), Moodle/BigBlueButton/Scalelite entegrasyonları, Sentinel ile yüksek erişilebilirlik ve izleme/doğrulama.

## Redis Nedir?

**Redis** (REmote DIctionary Server), veriyi **bellekte (RAM) tutan, açık kaynaklı, anahtar–değer (key-value) tabanlı** bir veri yapısı sunucusudur. Çok düşük gecikmeli (< 1 ms) okuma/yazma sağlar ve ilişkisel veritabanlarının yükünü azaltmak için **önbellek (cache)**, **oturum (session) deposu**, **mesaj kuyruğu** ve **kanal yayını (Pub/Sub)** olarak kullanılır.

- **Veri yapıları:** string, hash, list, set, sorted set, stream — düz KV'nin ötesinde zengin yapılar.
- **Pub/Sub:** mesajlaşma veriyolu — BBB bileşenlerinin haberleşmesi buna dayanır.
- **Kalıcılık (persistence):** RDB anlık görüntü + AOF (append-only file); sunucu yeniden başlasa da veri kaybolmaz (§3).
- **Bellek sınırı + eviction:** `maxmemory` ve politikası; hangi rol için neyin doğru olduğunu §3 ve aşağıdaki tablo belirler.
- **Tek-thread komut modeli:** komutlar sırayla işlenir; ölçekleme/HA shard ya da **Sentinel/Cluster** ile yapılır (§7).

> Kısaca Moodle/BBB/Scalelite yığınında Redis bir **veritabanı değil, hız ve haberleşme katmanıdır**: cache/session ile gecikmeyi, Pub/Sub ile bileşenler arası mesajlaşmayı, kuyruk ile asenkron iş yükünü üstlenir.

---

Redis bu yığında **üç ayrı rolde** kullanılır ve her birinin davranışı farklıdır — en kritik ayrım aşağıdadır:

| Rol | Kim kullanır | Doğru yapılandırma |
| :--- | :--- | :--- |
| **Mesaj veriyolu (message bus)** | BigBlueButton (`bbb-web`, `bbb-html5`, Akka, kayıt scriptleri) | **Kalıcı + tahliye YOK** (`appendonly yes`, `maxmemory-policy noeviction`) |
| **Oturum + uygulama önbelleği (session/MUC)** | Moodle | TTL/eviction olabilir (`volatile-lru` / `allkeys-lru`), kalıcılık opsiyonel |
| **Kuyruk + cache** | Scalelite | Kalıcılık önerilir (`appendonly yes`), kuyruk verisi kaybolmamalı |

> [!IMPORTANT]
> **Aynı Redis instance'ını BBB message bus ile Moodle session için PAYLAŞTIRMAYIN.** BBB veriyolu tahliye (eviction) kabul etmez; `allkeys-lru` gibi bir politika BBB'nin Pub/Sub/akış anahtarlarını silebilir → toplantılar bozulur. İdeal: hizmet/servis başına ayrı instance (en azından ayrı **`database`** numaraları değil — eviction politika instance düzeyindedir, ayrı instance gerekir).

---

## İçindekiler
- [Redis Nedir?](#redis-nedir)
1. [Kurulum](#1-kurulum)
2. [Güvenlik ve Sertleştirme](#2-güvenlik-ve-sertleştirme)
3. [Bellek, Eviction ve Kalıcılık](#3-bellek-eviction-ve-kalıcılık)
4. [Moodle Entegrasyonu (session + MUC)](#4-moodle-entegrasyonu-session--muc)
5. [BigBlueButton'da Redis (message bus)](#5-bigbluebuttonda-redis-message-bus)
6. [Scalelite'ta Redis](#6-scaleliteda-redis)
7. [Yüksek Erişilebilirlik — Sentinel](#7-yüksek-erişilebilirlik--sentinel)
8. [İzleme ve Loglar](#8-izleme-ve-loglar)
9. [Doğrulama / Test Komutları](#9-doğrulama--test-komutları)
10. [Kontrol Listesi](#10-kontrol-listesi)

> Not: Sunucu ve Sentinel yapılandırma örnekleri ilgili bölümlerde (§1–§3, §7) **satır içi** verilmiştir (ayrı config dosyası yoktur).

---

## 1. Kurulum

Ubuntu 22.04/24.04:

```bash
sudo apt update
sudo apt install -y redis-server
sudo systemctl enable --now redis-server
sudo systemctl status redis-server
redis-cli --version
```

**Moodle için PHP eklentisi** (ayrıca kurulur): `php8.3-redis` (bkz. `Moodle/02_Sunucu_Kurulumu_ve_Gereksinimler.md`).

Varsayılan yapılandırma dosyası `/etc/redis/redis.conf`'tir. Aşağıda **sertleştirilmiş, role-bağımsız bir çekirdek şablon** verilmiştir; `bind`/`requirepass`/`maxmemory` değerlerini ve **eviction politikasını rolünüze göre** (§3) düzenleyin:

```conf
# /etc/redis/redis.conf  (sertleştirilmiş çekirdek şablon)
bind 127.0.0.1 -::1
protected-mode yes
port 6379
requirepass COK_GUCLU_SIFRE
rename-command FLUSHALL ""
rename-command FLUSHDB  ""
maxmemory 2gb
maxmemory-policy noeviction     # BBB bus / Scalelite kuyruğu; Moodle session için: volatile-lru (§3)
appendonly yes
appendfsync everysec
save 900 1
dir /var/lib/redis
logfile /var/log/redis/redis-server.log
```

> Üretimde TLS (Redis 6+) tercih edin: `tls-port 6380` + `port 0`, `tls-cert-file`/`tls-key-file`/`tls-ca-cert-file` (bkz. `Certs/Sertifika_Rehberi.md`).

---

## 2. Güvenlik ve Sertleştirme

Redis varsayılan olarak **kimlik doğrulamasız** ve yanlış ayarlanırsa dışa açık çalışabilir. Zorunlu kontroller:

```conf
# /etc/redis/redis.conf
bind 127.0.0.1 10.0.0.30      # yalnizca loopback + bu node'un uygulama subnet'indeki iç IP'si
protected-mode yes             # bind dışı adreslerden gelen bağlantıyı reddeder
requirepass ÇOK_GUCLU_SIFRE    # kimlik doğrulama (mutlaka)
rename-command FLUSHALL ""     # yıkıcı komutları kapat (opsiyonel, kurumsal politika)
rename-command FLUSHDB  ""
```

> [!WARNING]
> Redis'i asla **herkese açık (0.0.0.0) + şifresiz** bırakmayın. İnternete açık şifresiz Redis, uzaktan kod çalıştırmaya kadar giden bir SPOF/giriş noktasıdır. Ağda erişimi güvenlik grubu/UFW ile uygulama node'larına kısıtlayın: `sudo ufw allow from 10.0.0.0/24 to any port 6379 proto tcp`.

**Servisler arası erişim** (Moodle/BBB/Scalelite ayrı sunuculardaysa): `requirepass` kullanın ve uygulamalar `auth` uygulasın (Moodle: `session_redis_password`; Scalelite: `REDIS_URL` içine gömün — bkz. §6). Üretimde mümkünse **TLS** (Redis 6+ `tls-port`) veya yalnızca özel ağ/VPN üzerinde çalıştırın.

---

## 3. Bellek, Eviction ve Kalıcılık

```conf
# /etc/redis/redis.conf — role'e göre seç:
maxmemory 2gb                     # Redis'e izin verilen tavan (RAM'in bir kısmı)

# BBB message bus / Scalelite kuyruğu İÇİN:
maxmemory-policy noeviction       # anahtar silme; dolarsa yazma hata verir (kapasite artır)

# Moodle session/MUC İÇİN (ayrı instance'ta):
maxmemory-policy volatile-lru     # TTL'i olanları önce tahliye et (cache için uygun)

# Kalıcılık:
appendonly yes                    # AOF — message bus & kuyruk için ŞART
appendfsync everysec
save 900 1                        # RDB yedek (ek güvence)
```

> [!IMPORTANT]
> **BBB `maxmemory-policy noeviction` + `appendonly yes` ister.** Pub/Sub ve kayıt/olay anahtarlarının sessizce silinmesi toplantı/medya/kayıt akışını kırar. `allkeys-*` politikalarını BBB instance'ında kullanmayın. Moodle session instance'ında `volatile-*`/`allkeys-lru` güvenlidir.

---

## 4. Moodle Entegrasyonu (session + MUC)

**4.1) Oturum (session) Redis'te** — `config.php` (Moodle 5.x, `/public` mimarisi):

```php
$CFG->session_handler_class = '\core\session\redis';
$CFG->session_redis_host     = '127.0.0.1';
$CFG->session_redis_port     = 6379;
$CFG->session_redis_auth     = 'ÇOK_GUCLU_SIFRE';   // requirepass varsa
$CFG->session_redis_database = 0;
$CFG->session_redis_prefix   = 'mdl_sess_';
$CFG->session_redis_acquire_lock_timeout = 120;
$CFG->session_redis_lock_expire          = 7200;
```

**4.2) Uygulama önbelleği (MUC) Redis** — arayüzden:
`Site administration > Plugins > Caching > Configured cache stores > Add instance (Redis)` → `127.0.0.1:6379` → **Application cache** için varsayılan sürücüyü Redis yapın. Detaylı akış: `Moodle/03_Veritabani_ve_Caching_Redis.md`.

> [!NOTE]
> Çoklu web node'ta Redis session **zorunludur** (sticky/`ip_hash` yerine); böylece node çökse de oturum kaybolmaz — bkz. `Moodle/10_Performans_ve_Olceklendirme.md`.

---

## 5. BigBlueButton'da Redis (message bus)

- BBB'nin **kendi node'undaki** Redis'i bileşenler arası **mesaj veriyoludur** (`bbb-web` ↔ `bbb-html5`/Meteor ↔ `bbb-apps-akka` ↔ kayıt scriptleri), Pub/Sub ile. Mimari rol: `BigBlueButton/01_BBB_Mimari_ve_Bilesenler.md`.
- **Yapılandırma yeri:** `/etc/redis/redis.conf` (bind/requirepass) ve BBB'nin Redis'e erişimi genelde `localhost` + `bigbluebutton.properties`/`production.yml` içindeki Redis ayarları ile.
- BBB'yi **Scalelite arkasında** çalıştırıyorsanız bu Redis **node-lokaldır** ve kümeleştirilmez; HA node düzeyinde gelir (bkz. `Integrations/05_High_Availability_and_Failover.md` §5.1). Yani BBB Redis'i için Sentinel **gerekmez** — node ölürse zaten o node'un toplantıları sonlanır.

> [!TIP]
> BBB Redis'inde eviction'ı kapatın (§3). Kurulum script'inin (`bbb-install.sh`) Redis'i doğru ayarladığını doğrulamak için: `redis-cli -a <sifre> CONFIG GET maxmemory-policy` → `noeviction` olmalı.

---

## 6. Scalelite'ta Redis

Scalelite, Redis'i **iş kuyruğu (Sidekiq/queue) ve cache** için kullanır. Scalelite `.env` içinde `REDIS_URL` tanımlanır (bkz. `Integrations/04_Scalelite_Balancer.md`):

```env
# Yerel/tek Redis
REDIS_URL=redis://:COK_GUCLU_SIFRE@10.0.0.30:6379/0

# Sentinel ile (bkz. §7)
REDIS_URL=redis://mymaster
REDIS_SENTINEL_HOSTS=sen1:26379,sen2:26379,sen3:26379
REDIS_SENTINEL_MASTER=mymaster
```

> [!WARNING]
> Scalelite kuyruğu kaybolursa bekleyen kayıt/işler yitirilir → `appendonly yes` ve tercihen Sentinel ile HA. `REDIS_URL` içindeki şifreyi düz metin `.env`'de tutuyorsanız dosyayı `chmod 600` yapın ve Git'e commit etmeyin.

---

## 7. Yüksek Erişilebilirlik — Sentinel

**Paylaşılan ve durumlu** Redis'ler (Scalelite kuyruğu, Moodle session) SPOF'tur; **Redis Sentinel** master düşünce bir replica'yı otomatik yeni master yapar. Önerilen: **3 sentinel + 1 master + 2 replica** (tek sayı/quorum).

Kurulum adımları (özet):
1. Master + replica'ları `redis.conf` ile kurun; replica'larda `replicaof <master_ip> 6379`.
2. Her sentinel düğümünde `/etc/redis/sentinel.conf` oluşturun (aşağıdaki şablon) ve `monitor mymaster <master_ip> 6379 2` satırındaki IP'yi düzenleyin.
3. Sentinel'leri başlatın, master'ı doğrulayın, sonra Scalelite/Moodle bağlantısını Sentinel'e veya Sentinel'i izleyen bir VIP/proxy'ye çevirin.

Örnek `/etc/redis/sentinel.conf`:

```conf
port 26379
daemonize no
logfile /var/log/redis/sentinel.log
dir /var/lib/redis
sentinel monitor mymaster 10.0.0.30 6379 2        # 2 = quorum (3 sentinel)
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
sentinel auth-pass mymaster COK_GUCLU_SIFRE        # master requirepass ile aynı
sentinel resolve-hostnames yes
```

Başlatma ve doğrulama:

```bash
redis-server /etc/redis/sentinel.conf --sentinel
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster   # kim master?
```

> [!IMPORTANT]
> **Moodle'un yerleşik Redis store'u Sentinel'i doğrudan desteklemez** (tek uç nokta bekler). Bu yüzden Moodle'a Sentinel'in kendisini değil, **aktif master'ı izleyen bir VIP/proxy** gösterin (keepalived + Sentinel `notification-client`, veya Sentinel-farkında proxy). Scalelite (redis-rb) ise `REDIS_SENTINEL_*` ile native destekler (§6). Tam HA topolojisi: `Integrations/05_High_Availability_and_Failover.md` §5.8.

---

## 8. İzleme ve Loglar

- **Loglar:** `/var/log/redis/redis-server.log` (ve sentinel logları). Merkezi toplama: `Integrations/03_Merkezi_Log_Yonetimi.md` (Promtail/Filebeat).
- **Metrikler:** `redis_exporter` (Prometheus) → `used_memory`, `evicted_keys`, `connected_clients`, `master_link_status`, `aof_enabled`.
- **Kritik alarmlar:**
  - `used_memory > maxmemory * 0.9` → kapasite uyarısı (noeviction'da yazma hatası gelir).
  - `evicted_keys > 0` BBB instance'ında → **yanlış politika** (noeviction olmalıydı).
  - `master_link_status:down` → Sentinel/replica kopuk.

```bash
redis-cli -a <sifre> INFO memory | grep -E "used_memory_human|maxmemory_human"
redis-cli -a <sifre> INFO replication
redis-cli -a <sifre> INFO stats | grep evicted_keys
```

---

## 9. Doğrulama / Test Komutları

```bash
redis-cli -a <sifre> ping                       # PONG
redis-cli -a <sifre> CONFIG GET maxmemory-policy # BBB: noeviction / Moodle: volatile-lru
redis-cli -a <sifre> CONFIG GET appendonly        # yes (bus/kuyruk)
redis-cli -a <sifre> INFO clients                 # bağlı uygulamarı gör
# Moodle oturumlarını kontrol (prefix'e göre):
redis-cli -a <sifre> --scan --pattern 'mdl_sess_*' | head
```

---

## 10. Kontrol Listesi

- [ ] `requirepass` tanımlı; Redis dışa açık değil (`bind` + `protected-mode yes` + firewall)
- [ ] BBB instance: `maxmemory-policy noeviction` + `appendonly yes`
- [ ] Moodle instance (ayrı): `volatile-lru`/`allkeys-lru` + `maxmemory` ayarlı
- [ ] Scalelite: `REDIS_URL` doğru; şifre `.env`'de gizli (`chmod 600`)
- [ ] HA gerekiyorsa: 3 Sentinel (tek sayı), `sentinel.conf` doğru master IP; Moodle için VIP/proxy
- [ ] İzleme: `redis_exporter` + `used_memory/evicted_keys/master_link_status` alarmları
- [ ] `redis-cli ping` ve `CONFIG GET` kontrolleri geçti

---

### Çapraz referanslar (düz metin — yerel link yok)
- Moodle cache/session ayrıntısı: `Moodle/03_Veritabani_ve_Caching_Redis.md`
- Moodle kurulum/requirement: `Moodle/02_Sunucu_Kurulumu_ve_Gereksinimler.md`
- BBB mesaj veriyolu rolü: `BigBlueButton/01_BBB_Mimari_ve_Bilesenler.md`
- Scalelite `.env`: `Integrations/04_Scalelite_Balancer.md`
- Redis Sentinel HA topolojisi: `Integrations/05_High_Availability_and_Failover.md` (§5.8)
- Redis logları/almalar: `Integrations/03_Merkezi_Log_Yonetimi.md`
