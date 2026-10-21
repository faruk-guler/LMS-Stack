# Bölüm 5: Yüksek Erişilebilirlik (HA), Yedeklilik ve Failover — Kurumsal moodle-bbb-stack Mimarisi

Bu bölüm, **Moodle + BigBlueButton (BBB) + Scalelite** kümesinin tek bir sunucunun çökmesine rağmen ayakta kalmasını (High Availability), planlı/planısız kesintilerde **otomatik veya yarı-otomatik failover** yapmasını ve **RPO/RTO** hedeflerini nasıl tasarlayacağını anlatır.

> Bu rehber önceki bölümleri tamamlar: Ölçeklenme (Scalelite/worker havuzu) için Scalelite Yük Dengeleyici (`04_Scalelite_Balancer.md`) ve BBB 08 (`BigBlueButton/08_Performans_ve_Olceklendirme_Scalelite.md`); Moodle web katmanı yatay ölçekleme için Moodle 10 (`Moodle/10_Performans_ve_Olceklendirme.md`); izleme için BBB 09 (`BigBlueButton/09_Izleme_Loglama_ve_SorunGiderme.md`) ve Merkezi Log (`03_Merkezi_Log_Yonetimi.md`). Bu bölüm ise **durumlu servislerin (DB/Redis/nesne depolama) ve giriş noktalarının** yedeğini ele alır.

---

## 5.1 Önce Zihinsel Model: BBB "Shared-Nothing" Bir Mimaridir

Kurumsal HA tasarımının en sık yapılan hatası, BBB'nin **kendi iç Redis/MongoDB/PostgreSQL'ini** kümeleştirmeye çalışmaktır. Doğru yaklaşım:

- Bir BBB sunucusu **kendine yeten (self-contained / shared-nothing)** bir birimdir. O node'un `redis`, `mongod` ve `bbb-web` PostgreSQL'i **yalnızca o node'taki aktif toplantıları** tutar.
- Bu node çökerse: yalnızca **o node'taki toplantılar** kaybolur. Platformun geneli etkilenmez. **Scalelite**, periyodik sağlık yoklamasında (poller) node'u `offline` işaretler ve yeni toplantıları sağlıklı node'lara dağıtır (04 (`04_Scalelite_Balancer.md`)).
- Sonuç: BBB **node bazında** ölçeklenir ve yedeklenir (N+1), **iç servisleri cluster'lanmaz**.

> [!IMPORTANT]
> Gerçek **tek hata noktaları (SPOF)**, BBB node'larının içi değil; **paylaşılan durumlu servislerdir**: Scalelite'ın dış **PostgreSQL + Redis**'i, Moodle'ın **PostgreSQL + Redis**'i, **kayıt deposu** (NFS/S3) ve **giriş noktaları** (Scalelite URL, Moodle URL, TURN). HA çalışmasının ağırlığı buraya gider.

---

## 5.2 Erişim Hedefleri (SLA / RPO / RTO)

Kurumsal bir moodle-bbb-stack için tipik hedefler (kurum politikasına göre ayarlayın):

| Hedef | Önerilen Değer | Anlamı |
| :--- | :--- | :--- |
| **Erişilebilirlik SLA** | %99,9 (ayda ~43 dk) — kritik kurum %99,95 | Yıllık/aylık kesinti bütçesi |
| **RPO (veri kaybı)** | DB ≤ 1 dk, kayıtlar ≤ 5 dk | Felaket senaryosında kabul edilebilir veri kaybı |
| **RTO (toparlanma)** | Servis ≤ 5 dk (otomatik failover), tam DR ≤ 30 dk | Kesintiden sonra hizmetin geri gelme süresi |
| **Kapaside kapasite** | **N+1** (havuzda en az 3 BBB; biri çökse kapasite yeter) | Pik trafik + tek node arızası marjı |

> [!NOTE]
> Sınav/kanlı yayın gibi **pik** senaryolarında hedef "kesintisiz" değil, **"göze batacak şekilde kademeli"** olmalıdır: Scalelite dolduğunda yeni toplantıları reddetmek yerine mevcut node'ları zorlamamalı; kapasite alarmı (Prometheus) önceden uyarmalıdır.

---

## 5.3 Kurumsal Referans Mimari (HA Topolojisi)

```text
                        ┌───────────────────────────────────────────┐
   Kullanıcı / LMS ───► │  VIP (keepalived)  ──►  HAProxy/Nginx ×2   │  (giriş katmanı)
                        └───────────────┬───────────────────────────┘
            https://bbb.kurum.edu  ─────┤       https://moodle.kurum.edu
                                        ▼
        ┌───────────────────────┐             ┌───────────────────────────┐
        │  Scalelite ×2 (stateless)│          │  Moodle Web Node ×N (stateless)│
        │  (keepalived ile active/  │          │  (Nginx+PHP-FPM; Redis session) │
        │   active veya active/pass) │          └──────────────┬────────────────┘
        └───────────┬───────────┘                             │
                    │                                          │
     ┌──────────────┴───────────────┐              ┌──────────┴───────────────┐
     ▼                              ▼              ▼                          ▼
┌──────────────┐  ┌──────────────┐ ┌───────────────┐         ┌────────────────────────┐
│ Postgres     │  │ Redis        │ │ Coturn TURN ×2│         │ Paylaşılan Depolama    │
│ HA (Patroni) │  │ Sentinel ×3  │ │ (VIP/DNS)     │         │ NFS+DRBD / S3 / Gluster │
│ 1P + N repl. │  │              │ │               │         │ (moodledata + BBB kayıtları)│
└──────────────┘  └──────────────┘ └───────────────┘         └────────────────────────┘
                    ▲
                    │ Scalelite health-poll (offline/online)
        ┌───────────┴───────────────────────────────┐
        │  BBB Worker Node 01 │ 02 │ 03 │ ... (N+1)  │   ← her node kendi redis/mongo/pg
        └────────────────────────────────────────────┘      (shared-nothing, cluster DEĞİL)
```

**Katmanlar:** Giriş (VIP+LB) → Durumsuz uygulama (Scalelite, Moodle web, BBB worker) → Durumlu çekirdek (Postgres HA, Redis Sentinel, nesne depolama) → Yardımcı (Coturn HA) → İzleme/uyarı.

---

## 5.4 SPOF Analizi ve Giderme Matrisi

| Bileşen | Tek başına SPOF mu? | HA Çözümü | Bu dokümanda |
| :--- | :--- | :--- | :--- |
| BBB worker node | Node ölürse **oradaki toplantılar** düşer (platform değil) | N+1 node + Scalelite health-poll | 04 (`04_Scalelite_Balancer.md`) |
| Scalelite uygulaması | **Evet** — düşerse tüm havuz erişilemez | **Stateless**; 2+ Scalelite + keepalived VIP (active/passive veya active/active) | §5.6 |
| Scalelite PostgreSQL | Evet | Streaming replication + **Patroni** otomatik failover | §5.7 |
| Scalelite Redis | Evet | **Redis Sentinel** (veya managed Redis) | §5.8 |
| Moodle PostgreSQL | Evet | Streaming replication + Patroni | §5.7 |
| Moodle Redis (session/MUC) | Evet | Sentinel + (Moodle tek-endpoint bağlantısı için VIP/proxy) | §5.8 |
| moodledata / BBB kayıtları (NFS) | Evet | Senkron replike (DRBD) veya **S3/obje depolama** versiyonlama | §5.9 |
| Coturn (TURN) | Evet (medya bağlanamaz) | 2+ Coturn + VIP/DNS round-robin | §5.10 |
| Giriş (DNS/TLS) | Evet | Anycast/multi-A + HSTS + sertifika otomasyonu | §5.11 |
| Let's Encrypt / cert | Kısmi | `certbot` systemd timer + dağıtık yenileme | BBB 10 (`BigBlueButton/10_Guncelleme_Yedekleme_ve_Guvenlik.md`) |

---

## 5.5 HA için Sunucu Envanteri (Örnek Kurum Planı)

| Rol | Adet | Not |
| :--- | :--- | :--- |
| Gateway/LB (HAProxy veya Nginx) + keepalived | 2 | VIP paylaşır (VRRP) |
| Scalelite (Docker Compose) | 2 | Stateless; aynı DB+Redis'i paylaşır |
| PostgreSQL (Patroni) | 3 | 1 primary + 2 standby (quorum) |
| Redis (Sentinel) | 3 | 1 master + 2 replica + 3 sentinel |
| MongoDB | — | Gerekli değil: BBB node'ları kendi mongo'sunu taşır |
| Coturn | 2 | VIP / DNS round-robin |
| Obje/NFS depolama | 2+ | DRBD senkron veya S3 (versioning) |
| BBB worker | N+1 | En az 3; her biri Ubuntu 24.04 + Docker (BBB 4.0) |
| Moodle web | N | PHP-FPM; session Redis'te |

> [!WARNING]
> **Quorum kuralı:** Patroni/etcd/Redis Sentinel **azınlık bölünmesini (split-brain)** önlemek için **tek sayıda** düğüm (3 veya 5) kullanır. 2 düğüm HA **değildir** — biri düşerse diğer yarı karar veremez.

---

## 5.6 Scalelite'ın Kendi HA'sı (Stateless → Çoklu Kurulum)

Scalelite uygulaması **durumsuzdur**: tüm durum dış **PostgreSQL** (meeting/recording metadata) ve **Redis**'te (kuyruk) tutulur. Bu yüzden aynı DB+Redis'e bağlanan **birden fazla Scalelite** instance'ı çalıştırabilirsiniz.

**Aktif/Pasif (en basit, önerilen başlangıç):** keepalived VIP'si tek bir Scalelite'te durur; öbürü "warm standby" olarak aynı compose'u ayağa kaldırmaya hazır bekler.

```nginx
# /etc/keepalived/keepalived.conf  (SCALELITE-01 — MASTER)
vrrp_instance VI_SCALELITE {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100                 # standby'da daha düşük (örn. 90)
    advert_int 1
    authentication { auth_type PASS  auth_pass SIR_DOSI }
    virtual_ipaddress { 203.0.113.10/32 }   # Scalelite VIP
    track_script { chk_scalelite }
}
# /etc/keepalived/keepalived.conf  (SCALELITE-02 — BACKUP)
#   ... state BACKUP, priority 90, aynı virtual_router_id + VIP
```

```bash
# /etc/keepalived/check_scalelite.sh  (health check)
#!/bin/bash
curl -fsS --max-time 5 http://127.0.0.1:3000/up   || exit 1
```

> [!NOTE]
> **Aktif/Aktif** isterseniz: VIP yerine DNS'e her iki Scalelite IP'sini round-robin verin veya önlerine HAProxy koyun; iki instance aynı Postgres+Redis'i paylaştığı için tutarlı çalışır (Scalelite yarışları Redis üzerinden senkronize eder).

Her Scalelite instance'ının `.env` dosyası **aynı** DB/Redis'i göstermelidir (bkz. 04 (`04_Scalelite_Balancer.md`) `.env`):

```env
# iki Scalelite'ta da BIREBIR ayni olacak (yerel DB degil, PAYLASILAN servise bak)
DATABASE_URL=postgres://scalelite:GIZLI@db-vip:5432/scalelite_production
REDIS_URL=redis://cache-vip:6379/0          # Sentinel icin §5.8'e bakın
SCALELITE_RECORDING_DIR=/var/bigbluebutton/spool   # ortak NFS/S3
```

---

## 5.7 PostgreSQL HA — Streaming Replication + Patroni (Otomatik Failover)

Hem **Scalelite** hem **Moodle** PostgreSQL'i tek sunucudan ibaretse, DB çöktüğünde platform tamamen durur. Kurumsal çözüm: **senkron/yarı-senkron streaming replication + Patroni** (otomatik leader election + failover).

```text
        Patroni (etcd/DCS quorum: node1,node2,node3)
   ┌───────────┬───────────┬───────────┐
   ▼           ▼           ▼
 primary     standby     standby
  (RW)     (R, senkron)   (R)   →  primary düşerse Patroni bir standby'ı ~10-30 sn'de promote eder
```

**1) Akış replikasyonu (replication kullanıcı + pg_hba):**

```sql
-- primary üzerinde
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'Gizli_Repl_Pas';
SELECT pg_create_physical_replication_slot('replica_01');
```

```text
# primary: pg_hba.conf — replikasyon bağlantılarına izin
host replication replicator 10.0.0.0/24 scram-sha-256
```

**2) Standby'ı `pg_basebackup` ile kurun (uygulamayı durdurmadan kopyalar):**

```bash
pg_basebackup -h primary-vip -U replicator -D /var/lib/postgresql/17/main \
  -P -R --slot=replica_01 --target=standby
```

**3) Patroni (kurumda otomatik failover için) — örnek `/etc/patroni/patroni.yml`:**

```yaml
scope: moodle-bb-db
name: pg1
restapi: { listen: 0.0.0.0:8008, connect_address: pg1.kurum.local:8008 }
etcd3: { hosts: etcd1.kurum.local:2379,etcd2.kurum.local:2379,etcd3.kurum.local:2379 }
bootstrap:
  dcs:
    postgresql:
      parameters: { synchronous_mode: true }   # RPO~0 için senkron rep
  initdb: [{ encoding: UTF8 }, { data-checksums }]
postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg1.kurum.local:5432
  data_dir: /var/lib/postgresql/17/main
  authentication:
    replication: { username: replicator, password: 'Gizli_Repl_Pas' }
    superuser:   { username: postgres,  password: 'Gizli_Super_Pas' }
```

```bash
patronictl -c /etc/patroni/patroni.yml list      # kim primary?
patronictl -c /etc/patroni/patroni.yml switchover moodle-bb-db pg1 pg2   # planlı geçiş
```

> Uygulama tarafı **sabit bir uç noktaya** bağlanmalı; bunu VIP (keepalived/Patroni VIP) veya **HAProxy/PGO** sağlar. Moodle bağlantısı için bkz. Moodle 10 (`Moodle/10_Performans_ve_Olceklendirme.md`); Scalelite `DATABASE_URL` yukarıda.

> [!TIP]
> `synchronous_mode: true` → yazma, standby'a düşmeden dönmez (RPO ≈ 0, hafif latency cezası). Yüksek yazma hacminde `synchronous_mode: false` (async) + kısa RPO kabulü daha gerçekcidir; kararı **RPO hedefinize** göre verin.

---

## 5.8 Redis HA — Sentinel (Master/Replica + Otomatik Failover)

**Scalelite Redis'i** (iş kuyruğu) kritik SPOF'tur. **Moodle Redis'i** (session/MUC cache) de. Standart çözüm **Redis Sentinel**: master düşerse sentinel bir replica'yı yeni master yapar.

**Topoloji:** 3 master/replica düğümü + 3 sentinel (quorum = 2).

> [!IMPORTANT]
> Redis Sentinel **yapılandırmasının tek kaynağı** artık `Redis/Redis_Rehberi.md` (§7, satır içi `sentinel.conf` şablonu dahil). Burada yalnızca HA bağlamındaki özet durur:
>
> - **Neden:** Scalelite kuyruğu ve Moodle session Redis'i SPOF'tur; Sentinel, master düşünce bir replica'yı otomatik yeni master yapar. **Topoloji:** 3 sentinel (quorum=2) + 1 master + ≥2 replica; `appendonly yes` şart (kuyruk verisi kaybolmasın).
> - **Scalelite:** `REDIS_SENTINEL_*` ile native Sentinel destekler.
> - **Moodle:** yerleşik store tek uç nokta bekler → Sentinel'i değil, **aktif master'ı izleyen bir VIP/proxy**'yi gösterin (keepalived + failover hook veya Sentinel-aware proxy). Kritik kuyruk için master'a `min-replicas-to-write 1`.

---

## 5.9 Paylaşılan Depolama HA (moodledata + BBB Kayıtları)

Hem Moodle `moodledata` hem BBB kayıtları (`/var/bigbluebutton/published`) **tek NFS sunucusundaysa** bu SPOF'tur. Seçenekler:

1. **NFS + DRBD (senkron blok replikasyonu):** iki depolama düğümü, active/passive; VIP failover. `moodledata` ve Scalelite spool aynı anda korunur.
2. **Ceph/GlusterFS (dağıtık):** çok düğüm, otomatik replikasyon (replica=3); tek sunucu düşse mount çalışmaya devam eder.
3. **S3 / obje depolama (önerilen, buluta uygun):** Scalelite `S3_BUCKET`+`S3_REGION` ile kayıtları S3'e yazar; Moodle `tool_objectfs` ile dosyaları S3'e taşır. **Versioning + lifecycle** ile silinme/bozulmaya karşı koruma.

```env
# Scalelite .env — S3 arşiv (NFS yerine/yanı sıra)
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET=bbb-kayitlari
S3_REGION=eu-central-1
```

> [!NOTE]
> Moodle tarafında `moodledata`'yı HA yapmak için `tool_objectfs` (S3) en temiz yoldur; böylece web node'ları paylaşımsız çalışır ve tek NFS darboğazı/kuyruğu kalkar. Detaylı paylaşımlı NFS kurulumu için Moodle 10 §10.3 (`Moodle/10_Performans_ve_Olceklendirme.md`).

---

## 5.10 Coturn (TURN) HA — Medya Bağlantısının Yedeği

Kısıtlı/NAT arkası ağlardan gelen kullanıcılar TURN'e bağımlıdır; **tek Coturn** düşerse bu kullanıcılar 1007/1020 alır (bkz. BBB 03 (`BigBlueButton/03_Ag_Guvenlik_Duvari_ve_TURN_STUN.md`)). HA:

- **2+ Coturn** sunucusu, **keepalived VIP** veya DNS round-robin (`turn.kurum.edu` → 2 A kaydı) ile.
- **Aynı `static-auth-secret`** tüm Coturn'larda ve BBB sunucularında (`bbb-install.sh -c <turn-host>:<secret>` ile verilen gizli) — böylece hangi TURN'e düşerse düşen HMAC doğrulanır.
- UDP port aralığı (3478 + relay 49152-65535) firewall'da her iki düğüm için açık.

```text
Coturn HA (keepalived) — TURN VIP 203.0.113.20
   ├── turn-01 (priority 100) ─┐
   └── turn-02 (priority  90) ─┘─► same static-auth-secret
```

> [!TIP]
> Coturn, Scalelite/BBB kadar "durumsuz" değildir (aktif relay oturumları) — failover anında **devam eden medya akışları** o TURN'ü yeniden kurar. Bu yüzden Coturn HA, "sıfır kesinti" değil **"hızlı yeniden bağlanma"** garantisi verir; kullanıcıyı 1020'den kurtarır.

---

## 5.11 Giriş / DNS / TLS Dayanıklılığı

- **Redundant DNS:** `bbb.kurum.edu` ve `moodle.kurum.edu` için **anycast veya birden çok A/AAAA** (multi-region). DNS health-check ile ölü VIP'yi yayından çıkarın.
- **TLS:** sertifika otomasyonu (`certbot` systemd timer / **acme.sh** + DNS-01 ile wildcard); süresi dolan sertifika = WebRTC/medya krizi (bkz. BBB 16 §16.5 (`BigBlueButton/16_Sik_Karsilasilan_Hatalar_ve_Kriz_Yonetimi.md`)).
- **HSTS + yönlendirme:** gateway HAProxy'de merkezî; tüm node'larda ayrı ayrı değil (tek noktadan yönetim).

---

## 5.12 İzleme Destekli Otomatik Failover (Prometheus/Alertmanager)

HA, **algılama** olmadan kötürümdür. BBB 09 (`BigBlueButton/09_Izleme_Loglama_ve_SorunGiderme.md`) ve Merkezi Log (`03_Merkezi_Log_Yonetimi.md`) üzerine şu HA-metrik alarmlarını ekleyin:

| Alarm | Koşul | Otomatik/Eylem |
| :--- | :--- | :--- |
| Scalelite down | `/up` 3× fail | Patroni/Sentinel/keepalived **otomatik**; operatör bildirimi |
| Postgres primary down | Patroni `no leader` | Patroni standby promote (RTO < 30 sn) |
| Redis master down | Sentinel `+switch-master` | Sentinel replica promote |
| BBB node offline | Scalelite poll fail | Scalelite havuzdan çıkarır; `bbb_node_down` alarm |
| Replication lag | `> 30s` (Postgres) | Capacity/planning alarmı (senkron modda yazma yavaşlar) |
| Disk (kayıt/NFS) %85 | eşik | Otomatik eski kayıt rotasyonu / S3'e taşıma |

Patroni ve Sentinel zaten **uygulama-müdahalesiz** promote yapar; Alertmanager → PagerDuty/Slack/Telegram ile **operatör bilgilendirmesi** + Grafana paneli gerekir.

---

## 5.13 Failover Runbook'ları (Acil Müdahale Kartları)

**Senaryo A — PostgreSQL primary çöktü:**
1. Patroni otomatik promote eder. Doğrula: `patronictl list` (yeni `Leader`).
2. Uygulama (Moodle/Scalelite) yeni primary VIP'ye bağlanmalı; takılı eski conn'ler için `SELECT pg_terminate_backend(...)` veya sadece LB'yi reload edin.
3. Eski primary düzeldiyse `patronictl reinit <scope> <eski-primary>` ile standby olarak geri alın.

**Senaryo B — Redis master çöktü:**
1. Sentinel promote eder: `redis-cli -p 26379 sentinel master mymaster` → yeni master IP.
2. Moodle VIP/proxy yeni master'a taşınmış olmalı; session kopmadıysa kullanıcı fark etmez.

**Senaryo C — Scalelite instance öldü:**
1. keepalived VIP diğer Scalelite'a geçer (~1-3 sn). Kullanıcı `bbb.kurum.edu`'yu yeniden açınca devam eder.
2. Ölen instance compose'u düzeltin; DB/Redis paylaşımlı olduğu için veri kaybı yok.

**Senaryo D — Bir BBB worker node çöktü:**
1. Scalelite onu `offline` işaretler; yeni toplantılar sağlıklı node'lara gider.
2. O node'daki toplantı katılımcıları arayüzden **yeniden bağlanır** (BBB paylaşımsızdır — o toplantının medyası kurtarılamaz, ama platform sağlamdır).
3. Node geri gelince Scalelite `online`'a çeker.

**Senaryo E — Coturn VIP düştü:**
1. İkinci Coturn VIP'i üstlenir; kısıtlı ağdaki kullanıcılar yeniden TURN'e bağlanır.

---

## 5.14 Felaket Kurtarma (DR) ve Yedekleme ile İlişkisi

HA = "ayağa kaldır"; **DR = "baştan kur"**. HA tek başına yeterli değildir; bölünmüş küme (split-brain), kötü güncelleme veya bölge çökmesi DR gerektirir:

- **DB:** `pg_basebackup` + WAL arşivi (PITR — zaman noktasına geri döndürme) + **haftalık tam + günlük artımlı**, ayrı bölgeye.
- **Kayıtlar/moodledata:** S3 versioning veya NFS→ikinci bölge asenkron replikasyon (bkz. §5.9).
- **Config:** tüm `.env`/`config.php`/`bigbluebutton.properties` **VaultSecrets/Ansible** ile sürümlü; asla yalnız diskte.
- **Rerun:** `bbb-install.sh` ile worker'ı ~15 dk'da yeniden kurabilirsiniz (bkz. BBB 02 (`BigBlueButton/02_Sistem_Gereksinimleri_ve_Kurulum.md`)) — HA'da node'lar **sığ (cattle)** olmalı, "inek (pet)" gibi tek tek sevmeyin.

> Moodle/BBB yedek betikleri için: Moodle 07 (`Moodle/07_Yedekleme_ve_Surum_Yukseltme.md`), BBB 10 (`BigBlueButton/10_Guncelleme_Yedekleme_ve_Guvenlik.md`).

---

## 5.15 HA Kurulum Kontrol Listesi

- [ ] En az **3 BBB worker** (N+1), Scalelite health-poll ile doğrulanmış
- [ ] **2 Scalelite** instance, aynı dış Postgres+Redis'e bağlı, **keepalived VIP** failover test edilmiş
- [ ] **PostgreSQL**: en az 1 primary + 1 senkron standby + **Patroni/etcd** (3 düğüm, quorum) — `switchover` + kill testi yapıldı
- [ ] **Redis Sentinel** (3 düğüm): Scalelite `REDIS_SENTINEL_*`; Moodle için **VIP/proxy** ile master'a bağlanıyor; persistans açık
- [ ] **Depolama**: moodledata + kayıtlar S3/DRBD/Ceph (tek NFS değil); `tool_objectfs` (Moodle) değerlendirildi
- [ ] **Coturn ×2** + VIP/DNS; `static-auth-secret` tüm düğümlerde aynı
- [ ] **Giriş**: redundant DNS + otomatik cert yenileme (wildcard önerilir)
- [ ] **Monitoring**: Prometheus + bigbluebutton-exporter + **Alertmanager** (HA alarmları §5.12 tanımlı)
- [ ] **DR**: PITR + çok bölgeli yedek; **tam restore testi** yapıldı (test edilmemiş yedek = yedek değildir)
- [ ] **Split-brain**: DB/etcd/sentinel tek sayı + quorum; DNS TTL kısa (failover'da eski VIP'ye yapışma yok)

---

> [!IMPORTANT]
> **Özet:** moodle-bbb-stack'te HA'nın kalbi *"BBB node'larını cluster'lamak" değil, "durumsuz uygulamayı (Scalelite/Moodle web/BBB worker) N+1 ölçeklendirip, DURUMLU çekirdeği (Postgres+Patroni, Redis+Sentinel, nesne depolama, Coturn, VIP'li giriş) yedeklemektir."* Bu ikisini karıştırmak, en pahalı kurumsal HA hatasıdır.

---

*Sonraki adım: bu mimarinin ölçülebilirliği için Merkezi Log Yönetimi (`03_Merkezi_Log_Yonetimi.md`) ve Prometheus/Grafana izleme (`BigBlueButton/09_Izleme_Loglama_ve_SorunGiderme.md`) bölümlerini HA alarmlarıyla tamamladığınızdan emin olun.*
