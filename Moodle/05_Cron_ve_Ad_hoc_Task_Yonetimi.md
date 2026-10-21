# 05 - Cron ve Ad-hoc Task Yönetimi

Moodle'ın "kalbi" Cron servisidir. Eğer Cron çalışmazsa; e-postalar gitmez, ödevler notlandırılmaz, kullanıcılar sistemden düşmez ve site zamanla şişerek yavaşlar.

## 5.1 CLI Cron Kurulumu (crontab)

Moodle cron'u asla bir web tarayıcısı üzerinden tetiklenmemelidir.

```bash
sudo crontab -u www-data -e
```

Satırı ekleyin:

```bash
* * * * * /usr/bin/php8.3 /var/www/moodle/admin/cli/cron.php >/dev/null 2>&1
```

> [!IMPORTANT]
> PHP sürüm yolunu (`/usr/bin/php8.3`) kesinlikle belirtin. Sistem varsayılan `php` komutu eski bir versiyonu gösteriyor olabilir. `which php8.3` ile doğrulayın.

## 5.2 systemd Timer Alternatifi (Önerilen)

`crontab` yerine systemd timer kullanmak, başarısız çalışmaları sistem loglarına (`journalctl`) kaydeder ve izlemeyi kolaylaştırır.

```bash
sudo nano /etc/systemd/system/moodle-cron.service
```

```ini
[Unit]
Description=Moodle Cron Job
After=network.target

[Service]
Type=oneshot
User=www-data
ExecStart=/usr/bin/php8.3 /var/www/moodle/admin/cli/cron.php
StandardOutput=journal
StandardError=journal
```

```bash
sudo nano /etc/systemd/system/moodle-cron.timer
```

```ini
[Unit]
Description=Moodle Cron Timer
Requires=moodle-cron.service

[Timer]
OnCalendar=minutely
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now moodle-cron.timer
sudo systemctl list-timers | grep moodle   # Çalışıyor mu kontrol et
```

## 5.3 Ad-hoc Tasks (Anlık Görevler)

Modern Moodle sürümlerinde bazı işlemler (kurs yedekleme, bildirim gönderimi) "Ad-hoc" olarak kuyruğa alınır. Cron her çalıştığında bu kuyruktan bir miktar işi eritir.

```bash
# Kuyruktaki bekleyen görev sayısını öğren
sudo -u www-data php8.3 /var/www/moodle/admin/cli/adhoc_task.php --list

# Kuyruğu manuel olarak hemen çalıştır
sudo -u www-data php8.3 /var/www/moodle/admin/cli/adhoc_task.php --execute

# Belirli bir görevi çalıştır (görev sınıf adıyla)
sudo -u www-data php8.3 /var/www/moodle/admin/cli/adhoc_task.php --classname="\core\task\backup_adhoc_task"
```

## 5.4 Cron Durumunu İzleme

`Site Administration > Server > Scheduled Tasks` sayfasında her görevin son çalışma zamanını ve sonucunu görebilirsiniz.

```bash
# Son cron / scheduled task çalışma zamanlarini CLI ile gor (cron.php --help desteklemez)
sudo -u www-data php8.3 /var/www/moodle/admin/cli/scheduled_task.php --list | head

# journalctl ile systemd timer loglarını incele
sudo journalctl -u moodle-cron.service --since "1 hour ago"
```

| Renk/Durum | Anlamı | Eylem |
| :--- | :--- | :--- |
| ✅ Yeşil | Son çalışma başarılı | İzlemeye devam |
| ⚠️ Sarı | Uzun süredir çalışmadı | Cron servisini kontrol et |
| ❌ Kırmızı | Hata ile sonlandı | Log detayına bak |
| 🔒 Kilitli | Görev kilitlenmiş | DB'den kilidi temizle |

## 5.5 Kilitlenen Görevleri Temizleme

Bazen bir görev (genellikle kurs yedekleme) yarıda kalırsa kilit (lock) bırakır ve bir daha çalışmaz.

```bash
# mdl_task_adhoc tablosunda asılı/kuyrukta biriken görevi tespit et (id + classname'i not edin)
sudo -u postgres psql moodledb -c "SELECT id, classname, faildelay, timetonext FROM mdl_task_adhoc ORDER BY timetonext;"

# Takılı kalan görevi sil — <ID> yerine yukarıdaki sorgudan gelen sayıyı yazın (id ile silmek en güvenlisidir)
sudo -u postgres psql moodledb -c "DELETE FROM mdl_task_adhoc WHERE id=<SILINECEK_ID>;"

# Ardından cache temizle ve Cron'u başlat
sudo -u www-data php8.3 /var/www/moodle/admin/cli/purge_caches.php
```

> [!WARNING]
> Kilidi temizlemeden önce görevin gerçekten asılı kaldığını doğrulayın. `Site Administration > Server > Scheduled tasks` sayfasından görevin "Running since" değerinin 1 saatten eski olup olmadığını kontrol edin.

## 5.6 Kurumsal E-posta (SMTP / Exchange) Yapılandırması

Cron'un başarıyla çalışmasıyla beraber forum bildirimlerinin ve şifre sıfırlama maillerinin gönderilebilmesi için kurumsal SMTP tanımlarının harici olarak yapılmış olması gerekir.

`Site Administration > Server > Email > Outgoing mail configuration` yolunu izleyin:

* **SMTP hosts:** `mail.kurum.local` (Exchange sunucu IP veya hostname)
* **SMTP port:** `587` (TLS) veya `25` (Güvenlik Yok)
* **SMTP Güvenlik:** `TLS` (587 için) veya `Yok` (25 için)
* **SMTP kullanıcı:** `moodle@kurum.com`
* **Gönderen adresi / adı:** `moodle@kurum.com` / `Moodle`

**Bilinmesi Gereken Kurumsal Exchange Kuralları:**

1. **Relay İzni:** Moodle sunucusunun IP adresine Exchange sunucusu üzerinde mutlaka Relay (Aktarım) izni verilmelidir, aksi takdirde mailler kabul edilmez.
2. **Firewall İzni:** Kurum ağındaki Firewall üzerinde 25 veya 587 portlarının Moodle sunucusundan Mail sunucusuna doğru açık olduğundan emin olun.

> [!NOTE]
> E-postaların spama düşmemesi için gereken SPF, DKIM ve DMARC kayıtlarının detayları için E-posta Altyapısı ve DMARC (`Integrations/02_Eposta_Altyapisi_ve_DMARC.md`) rehberine bakın.

---

> [!TIP]
> Cron çıktısını takip etmek istiyorsanız syslog entegrasyonu yerine görev bazlı loglama yapabilirsiniz. `Admin > Plugins > Logging > Standard Log` altından loglama seviyesini artırın.
