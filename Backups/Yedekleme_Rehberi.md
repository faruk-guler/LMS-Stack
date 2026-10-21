# Yedekleme ve Felaket Kurtarma (DR) Rehberi — LMS Stack

Bu dosya **LMS Stack genelinde yedekleme + felaket kurtarma işlemlerinin tek kaynağıdır** (single source of truth): ne yedeklenir, otomatik betikler, tutma (retention) politikası, PostgreSQL **PITR**, hedefler (yerel/off-site/S3), **doğrulama (test restore)** ve geri yükleme runbook'ları — Moodle + BigBlueButton + Scalelite + Redis + Greenlight için ortak.

> **Örtüşmeyi önleyen sınır (tek kaynak):** Yedekleme *yöntem/betik/DR* burada. Ürün kurulumları ilgili bölümlerde kalır (ör. Moodle `config.php` → `Moodle/02`, Scalelite kurulumu → `Integrations/04`). RPO/RTO ve HA (çalışan kümenin ayakta kalması) → `Integrations/05`; HA "ayağa kaldırır", **DR/yedek "baştan kurar"** — ikisi tamamlar.

## İçindekiler
1. [Golden Trio: eksik yedeğin üç ayağı](#1-golden-trio-eksik-yedeğin-üç-ayağı)
2. [RPO / RTO ve Retention](#2-rpo--rto-ve-retention)
3. [Ne Yedeklenir (hizmet bazında)](#3-ne-yedeklenir-hizmet-bazında)
4. [Otomatik Yedek Betikleri](#4-otomatik-yedek-betikleri)
5. [PostgreSQL Tam Yedek + PITR](#5-postgresql-tam-yedek--pitr)
6. [Hedefler, Şifreleme ve Off-site](#6-hedefler-şifreleme-ve-off-site)
7. [Doğrulama: Restore Drill (test edilmemiş yedek = yedek değildir)](#7-doğrulama-restore-drill)
8. [Geri Yükleme Runbook'ları](#8-geri-yükleme-runbookları)
9. [Zamanlama, Loglama, Alarm](#9-zamanlama-loglama-alarm)
10. [Güvenlik](#10-güvenlik)
11. [Kontrol Listesi](#11-kontrol-listesi)

---

## 1. Golden Trio: eksik yedeğin üç ayağı

Bir LMS/BBB yedeği **3 parçadan** oluşur; biri eksikse yedek **işe yaramaz**:

| Parça | İçerik | Araç |
| :--- | :--- | :--- |
| **Veritabanı** | Moodle/BBB/Greenlight/Scalelite Postgres, BBB MongoDB | `pg_dump` / `mongodump` / `pg_basebackup` |
| **Dosyalar** | `moodledata`, BBB `/var/bigbluebutton` (kayıtlar), `/var/www/bigbluebutton-default` | `tar` / `rsync` / S3 |
| **Yapılandırma** | `config.php`, `/etc/bigbluebutton/*`, `.env`, Nginx vhost, `redis.conf` | `tar` / yapılandırma deposu (Git — sırlarsız) |

> [!CAUTION]
> **DB ve dosyalar AYNI ANDA** alınmalı. Biri diğerinden eskiyse "Missing file / kayıp içerik" hataları oluşur. Yedek öncesi site **bakım moduna** alınmalı (Moodle: `admin/cli/maintenance.php --enable`).

---

## 2. RPO / RTO ve Retention

| Hedef | Öneri | Anlam |
| :--- | :--- | :--- |
| **RPO** | DB ≤ 1 dk (PITR/WAL arşivi), dosyalar ≤ 5 dk (S3 versioning) | Kabul edilebilir veri kaybı penceresi |
| **RTO** | Restore ≤ 30–60 dk (prosedür test edilmişse) | Hizmetin geri gelme süresi |
| **Retention** | 7 günlük günlük + 4 haftalık haftalık + 6–12 aylık aylık arşiv | "3-2-1" kuralı aşağıda |

**3-2-1 kuralı:** en az **3** kopya, **2** farklı medya, **1** kopya **off-site/offline** (ransomware/ bölge çökmesi koruması).

---

## 3. Ne Yedeklenir (hizmet bazında)

**Moodle**
- PostgreSQL veritabanı (`moodledb`)
- `moodledata` (filedir/cache; **dosya ağacı DB ile tutarlı** olmalı)
- `config.php` (sırlar — şifreli sakla), `moodledata/lang` (özel çeviriler), tema/eklenti özelleştirmeleri

**BigBlueButton** (sunucu yeniden-kurulabilir; kritik olan **kayıtlar ve ayarlar**)
- `/var/bigbluebutton/` (özellikle `published/` ve `unpublished/` kayıtlar) → **NFS/S3** varsa oradan
- `/etc/bigbluebutton/` (tüm override `.properties`/`.yml`), `/var/www/bigbluebutton-default/` (logo/default.pdf)
- **Greenlight v3** Postgres DB + `.env` (bkz. `BigBlueButton/10`, `16`)
- BBB'nin kendi Postgres'i (`bbb-record-core` DB) ve Redis AOF/RDB (kaynak/oturum)

**Scalelite**
- Harici **PostgreSQL** (`scalelite_production`) — meeting/recording metadata
- `SCALELITE_RECORDING_DIR` spool / S3 bucket (kayıt dosyaları)

**Redis (durum paylaşılan instance'larda)**
- `redis-cli BGSAVE` (RDB) + AOF dizini; **kaybolması** oturum/kuyruk kaybı demek (bkz. `Redis/Redis_Rehberi.md`)

---

## 4. Otomatik Yedek Betikleri

### 4.1 Moodle tam yedek (DB + moodledata + config)

```bash
#!/bin/bash
# moodle-backup.sh — Golden Trio yedeği. faruk-guler 2026 (örnek)
set -euo pipefail

BACKUP_DIR="/backup/moodle"; DATE=$(date +%Y-%m-%d_%H-%M)
DB_NAME="moodledb"; DB_USER="moodleuser"; DB_HOST="127.0.0.1"
MOODLE_ROOT="/var/www/moodle"; DATA_ROOT="/var/www/moodle/moodledata"
KEEP_DAYS=7

mkdir -p "$BACKUP_DIR/$DATE"
echo "[$(date)] Yedek basladi..."

# 1) Bakım modu (dosya+DB tutarlılığı için)
sudo -u www-data php8.3 "$MOODLE_ROOT/admin/cli/maintenance.php" --enable

# 2) Veritabanı (custom format = paralel + sıkıştırılmış)
PGPASSWORD="${DB_PASS:?DB_PASS env gerekli}" pg_dump -Fc -h "$DB_HOST" -U "$DB_USER" "$DB_NAME" \
  > "$BACKUP_DIR/$DATE/moodle-db.dump"

# 3) moodledata + config
tar -czf "$BACKUP_DIR/$DATE/moodledata.tar.gz" -C "$(dirname "$DATA_ROOT")" "$(basename "$DATA_ROOT")"
tar -czf "$BACKUP_DIR/$DATE/config.tar.gz" -C "$MOODLE_ROOT" config.php

# 4) Bakım modunu kapat
sudo -u www-data php8.3 "$MOODLE_ROOT/admin/cli/maintenance.php" --disable

# 5) Off-site kopya (rsync örnek)
rsync -a --delete "$BACKUP_DIR/" backup@offsite:/srv/backup/moodle/ || true

# 6) Retention temizliği
find "$BACKUP_DIR" -maxdepth 1 -type d -mtime +"$KEEP_DAYS" -exec rm -rf {} + 2>/dev/null || true
echo "[$(date)] Tamamlandi: $BACKUP_DIR/$DATE"
```

```bash
sudo chmod 700 /usr/local/bin/moodle-backup.sh
sudo mkdir -p /backup/moodle && sudo chown www-data:www-data /backup/moodle
# DB_PASS'ı /etc/moodle-backup.env (chmod 600) içinde tutun, betiğe source edin
echo "0 2 * * * root . /etc/moodle-backup.env && /usr/local/bin/moodle-backup.sh >> /var/log/moodle-backup.log 2>&1" \
  | sudo tee /etc/cron.d/moodle-backup
```

### 4.2 BBB kayıtları + config yedeği

```bash
#!/bin/bash
# bbb-backup.sh — kayıtlar + /etc/bigbluebutton (snapshot/temiz kuruma göre)
set -euo pipefail
TS=$(date +%Y-%m-%d_%H-%M); DST="/backup/bbb/$TS"; mkdir -p "$DST"
# Kayıtlar (paylaşımlı NFS/S3'e taşındıysa orayı hedefleyin):
rsync -a /var/bigbluebutton/published/ "$DST/published/"
rsync -a /var/bigbluebutton/unpublished/ "$DST/unpublished/" 2>/dev/null || true
# Ayarlar:
tar -czf "$DST/etc-bigbluebutton.tar.gz" -C / etc/bigbluebutton var/www/bigbluebutton-default
# Off-site / S3:
# aws s3 sync "$DST" s3://bbb-backup/$TS --sse AES256
```

> [!NOTE]
> BBB **sunucusu yeniden kurulabilir** (`bbb-install.sh`); asıl vazgeçilmez olan **kayıtlar** ve `/etc/bigbluebutton` ayarlarıdır. 3.x→4.0 geçişinde (bkz. `Integrations/05`) kayıtlar yeni sunucuya taşınır.

### 4.3 Greenlight / Scalelite Postgres dump

```bash
# Greenlight v3 (docker):
docker exec greenlight-db pg_dump -U postgres greenlight_production | gzip > /backup/greenlight/gl-$(date +%F).sql.gz
# Scalelite:
docker compose -f /opt/scalelite/docker-compose.yml exec -T scalelite-api \
  pg_dump "$DATABASE_URL" | gzip > /backup/scalelite/sl-$(date +%F).sql.gz
# Redisi:
redis-cli -a "$REDIS_PASS" BGSAVE    # RDB; AOF dizini ayrıca yedeklenir
```

---

## 5. PostgreSQL Tam Yedek + PITR

`pg_dump` tek başına anlıktır; **zaman noktasına geri dönüş (PITR)** için base backup + WAL arşivi gerekir:

```bash
# 1) Base backup (calisirken, -c checksum ile):
pg_basebackup -h 127.0.0.1 -U replicator -D /backup/pg/base -Ft -z -Xs -P -c l
# 2) WAL arşivi (postgresql.conf):
#    archive_mode = on
#    archive_command = 'test ! -f /backup/pg/wal/%f && cp %p /backup/pg/wal/%f'
#    (bkz: postgresql.conf ayarlari -> Database/Integrations/05 §5.7)
```

Geri yükleme (belirli bir ana kadar):

```bash
# 1) base backup'i data dizinine aç, 2) recovery.signal oluştur,
# 3) postgresql.conf/recovery ayarı:
restore_command = 'cp /backup/pg/wal/%f %p'
recovery_target_time = '2026-10-18 14:30:00'   # RPO hedefi
recovery_target_action = 'promote'
# pg_ctl restart -> PostgreSQL WAL replay ile o ana kadar getirir.
```

---

## 6. Hedefler, Şifreleme ve Off-site

- **Katman 1 (hızlı):** lokal `/backup` (NVMe) — son 7 gün, hızlı restore.
- **Katman 2 (off-site):** ikinci bölgeye `rsync` (SSH) veya **S3** (`--sse AES256`, **versioning + lifecycle**). `rclone` ile de taşınabilir.
- **Şifreleme:** yedekleri **at-rest** şifreli tutun (S3 SSE, veya `age`/`gpg` ile dosya şifreleme):
  ```bash
  tar -czf - "$DATA_ROOT" | age -r <alici-pub> -o moodledata.tar.gz.age
  ```
- **Değişmezlik (immutability):** S3 **Object Lock** / WORM — ransomware'in yedekleri silmesini engeller.

---

## 7. Doğrulama: Restore Drill

> [!IMPORTANT]
> **Test edilmemiş yedek, yedek değildir.** Ayda bir, izole bir VM/konteynerde "kuru çalışma" restore yapın:

```bash
# Örnek: Moodle restore drill (test ortaminda)
createdb -T template0 test_moodledb -E UTF8
# moodle-db.dump `pg_dump -Fc` (custom/binary) ile alindi -> gunzip GEREKMEZ, dogrudan pg_restore:
pg_restore -d test_moodledb /backup/moodle/latest/moodle-db.dump
# config.php'de dataroot/wwwroot'u test'e çevir, siteyi açip 1-2 ders + dosya indirmeyi dogrula.
```

Kontrol: DB bağlantısı ✓, örnek dosyalar `moodledata/filedir`'den iner ✓, BBB'de örnek kayıt oynatılır ✓, Greenlight admin girişi ✓.

---

## 8. Geri Yükleme Runbook'ları

**8.1 Moodle tam geri yükleme**
```bash
sudo -u www-data php8.3 /var/www/moodle/admin/cli/maintenance.php --enable
# config.php yedeği + moodledata + DB geri konur
gunzip -c /backup/moodle/$TS/moodledata.tar.gz | tar -x -C /var/www/moodle   # dataroot yerine
pg_restore -c -d moodledb --clean --if-exists /backup/moodle/$TS/moodle-db.dump
sudo -u www-data php8.3 /var/www/moodle/admin/cli/purge_caches.php
sudo -u www-data php8.3 /var/www/moodle/admin/cli/maintenance.php --disable
```
> Yerinde DB restore yerine **yeni sunucuya** taşıyacaksanız `config.php`'deki `$CFG->wwwroot`/`dataroot` ve DNS'i güncelleyin.

**8.2 BBB kayıtları** — `/var/bigbluebutton/published/` hedefe `rsync` + gerekirse `bbb-conf --restart`. Scalelite'ta merkezi S3/NFS zaten tek noktadır (bkz. `Integrations/04 §8`).

**8.3 Greenlight DB** (çöken/taşınan sunucu):
```bash
docker compose exec -T greenlight-db psql -U postgres -d greenlight_production \
  -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
gunzip -c /backup/greenlight/gl-YYYY-MM-DD.sql.gz | docker compose exec -T greenlight-db psql -U postgres -d greenlight_production
```

**8.4 Scalelite DB + Redis** — `DATABASE_URL` pointing Postgres'e restore, Redis AOF/RDB geri konur; `docker compose up -d`.

---

## 9. Zamanlama, Loglama, Alarm

- **Zamanlama:** cron (`/etc/cron.d/*`) veya **systemd timer** (log → `journalctl`, izleme kolay).
- **Loglama:** `/var/log/moodle-backup.log`; başarısızlıkta alarm.
- **Başarı sinyali metrik:** Prometheus `node_exporter` textfile veya `pushgateway` ile `backup_last_success_timestamp_seconds` yayınlayın; **`time() - metric > eşik` → alarm**. Detay: `Integrations/03`.
- **Rapor:** haftalık "restore drill geçti/kaldı" kaydı tutun.

---

## 10. Güvenlik

- Yedekler **kişisel veri** (öğrenci notları, e-posta) içerir → **şifreli** sakla, erişimi en az yetkiye indir (off-site'te de).
- `config.php`/`.env` yedekleri **düz Git deposuna commit edilmez**; `.gitignore` + sır yönetimi (bkz. `Security/Guvenlik_Rehberi.md` §7).
- Yedek klasör izinleri: `chmod 700`, sahiplik `root`/dedicated backup kullanıcısı.
- Silme/rotasyon betikleri `set -e` + doğru hedef yolu doğrular (yanlış dizine `rm -rf` felaketini önler).

---

## 11. Kontrol Listesi

- [ ] Golden Trio (DB + dosya + config) tek betikte, **bakım modunda** alınıyor
- [ ] Günlük + haftalık + aylık retention; **3-2-1** ve off-site/S3 versioning + Object Lock
- [ ] PostgreSQL **PITR** (base backup + WAL arşivi) kurulu
- [ ] BBB **kayıtlar** + `/etc/bigbluebutton`, Greenlight/Scalelite **Postgres** dump'ları alınıyor
- [ ] Yedekler **şifreli**; `.env`/`config.php` sırları Git'te YOK
- [ ] **Aylık restore drill** yapılıyor ve kaydediliyor
- [ ] `backup_last_success` metriği + alarm tanımlı (log/izleme → `Integrations/03`)
- [ ] RPO/RTO hedefleri dokümante ve HA (`Integrations/05`) ile uyumlu

---

### Çapraz referanslar (düz metin — yerel link yok)
- Ürün kurulum/özellik ayarları: `Moodle/02`, `Moodle/04` (moodledata), `Moodle/07` (sürüm yükseltme/rollback), `BigBlueButton/10`
- Scalelite kayıt/NFS-S3: `Integrations/04` · Greenlight DB restore krizi: `BigBlueButton/16`
- HA (canlı küme) & PITR DB failover: `Integrations/05` · Redis kalıcılık: `Redis/Redis_Rehberi.md`
- Sır güvenliği: `Security/Guvenlik_Rehberi.md` §7 · İzleme/almalar: `Integrations/03`
