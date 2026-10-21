# 07 - Sürüm Yükseltme (Upgrade) ve Rollback (Moodle 5.3 LTS — Git Yöntemi)

Moodle'ı arayüzden "Güncelle" butonuna basarak yükseltmek, büyük veritabanlarında felaketle sonuçlanabilir. Sistem yöneticileri her zaman **CLI** ve **Git** kullanır.

> [!NOTE]
> **Moodle 5.x Yükseltme Yolu:** Güncel bir 4.x sürümünden (özellikle **4.5 LTS**) doğrudan 5.x'e yükseltme desteklenir. Daha eski bir sürümedeyseniz önce **4.5 LTS**'ye, ardından **5.3 LTS**'ye geçin. Her majör geçişten önce mutlaka tam yedek alın ve bir test ortamında deneme yükseltmesi yapın.

## 7.1 Yedekleme (tek kaynak: `Backups/Yedekleme_Rehberi.md`)

Moodle'un **Golden Trio** yedeği (PostgreSQL + `moodledata` + `config.php`), otomatik `moodle-backup.sh` betiği, cron zamanlama, retention, **PITR** ve geri yükleme prosedürü artık **`Backups/Yedekleme_Rehberi.md`** içinde (§3, §4.1, §8.1). Bu bölüm yalnızca **sürüm yükseltme / rollback**'a odaklanır.

> [!IMPORTANT]
> Her yükseltme öncesi tam yedek alın (`Backups` rehberi §8.1 restore'u bildiğinizden emin olun). Aşağıdaki adımlar dosya/DB yedeğinin **alınmış olduğunu** varsayar.

## 7.2 Git ile Sürüm Yükseltme (Upgrade)

> [!WARNING]
> Yükseltme öncesi mutlaka yedek alın. Yükseltme sonrası geri dönmek için `git checkout` ve veritabanı restore gerekecektir.

```bash
# 1. Mevcut Moodle sürümünü kontrol et
cat /var/www/moodle/version.php | grep "\$version" | head -1

# 2. Bakım modunu aç
sudo -u www-data php8.3 /var/www/moodle/admin/cli/maintenance.php --enable

# 3. Git ile yeni sürüme geç
cd /var/www/moodle
sudo git fetch origin
sudo git checkout MOODLE_503_STABLE    # Güncel 5.3 LTS dalı

# 4. Cache'i temizle
sudo -u www-data php8.3 admin/cli/purge_caches.php

# 5. Veritabanı şemasını güncelle
sudo -u www-data php8.3 admin/cli/upgrade.php --non-interactive

# 6. Bakım modunu kapat
sudo -u www-data php8.3 admin/cli/maintenance.php --disable
```

## 7.3 Yükseltme Sonrası Doğrulama

```bash
# Tüm eklentilerin güncel olduğunu kontrol et
sudo -u www-data php8.3 admin/cli/upgrade.php --non-interactive

# Yeni sürüm numarasını doğrula
cat /var/www/moodle/version.php | grep "\$release"

# Site yönetim panelinde bildirim varsa
# Site Administration > Notifications sayfasını ziyaret edin
```

## 7.4 Rollback (Geri Dönüş) Stratejisi

```bash
# 1. Git ile eski commit'e geri dön
cd /var/www/moodle
sudo git log --oneline -5                     # Eski commit hash'ini bul
sudo git checkout <eski_commit_hash>

# 2. Veritabanını yedekten geri yükle
sudo -u www-data php8.3 admin/cli/maintenance.php --enable
gunzip -c /backup/moodle/2026-03-01_02-00/moodle-db.sql.gz | PGPASSWORD="sifre" psql -U moodleuser moodledb

# 3. moodledata'yı yedekten geri yükle (gerekirse)
sudo -u www-data php8.3 admin/cli/maintenance.php --disable
```

---

> [!TIP]
> Yedek betiklerinin başarıyla çalıştığını doğrulamak için `/var/log/moodle-backup.log` dosyasını düzenli kontrol edin. Bir monitoring aracıyla (Zabbix, Nagios) log dosyasındaki hata kelimelerini izlemek daha güvenlidir.
