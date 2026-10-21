# 12 - Analitik ve Raporlama (Learning Analytics)

Moodle sadece veri toplamaz; bu verileri kullanarak "risk altındaki öğrencileri" tahmin edebilir ve yönetime anlamlı raporlar sunabilir.

## 12.1 Log Store (Log Mağazası) Yapılandırması

Moodle'da her kullanıcı eylemi (sayfa görüntüleme, dosya yükleme, sınav girişi) varsayılan olarak **Standard Log** tablosuna yazılır.

```text
Site Administration > Plugins > Logging > Manage log stores
```

| Log Store | Avantaj | Dezavantaj |
| :--- | :--- | :--- |
| Standard Log (DB) | Hazır, kolay sorgulanabilir | Büyük veritabanı, yavaşlar |
| Legacy Log | Eski eklentilerle uyumluluk | Artık önerilmez |
| External Logstore (Elasticsearch) | Çok hızlı, ölçeklenebilir | Ek altyapı gerektirir |

> [!NOTE]
> 5000+ kullanıcılı sistemlerde `mdl_logstore_standard_log` tablosu kısa sürede milyonlarca satıra ulaşır. Belirli aralıklarla (`Site Admin > Reports > Log` sayfasından) eski logları temizleyin veya bir "Log lifetime" ayarı belirleyin.

## 12.2 Learning Analytics Modelleri

Moodle 3.4+ sürümünden itibaren yerleşik makine öğrenmesi modelleri bulunmaktadır.

```text
Site Administration > Analytics > Analytics models
```

| Model | Açıklama | Gereksinim |
| :--- | :--- | :--- |
| Students at risk of dropping out | Dersi bırakma riski tahmini | Geçmiş kayıt verisi |
| No teaching in forum | Eğitmen tartışma forumunu ihmal ediyor | Forum aktivitesi |
| Upcoming activities due | Yaklaşan teslim tarihleri hatırlatması | Gelecek aktiviteler |

```bash
# Modelleri CLI ile eğit (büyük sistemlerde cron yerine manuel çalıştırın)
sudo -u www-data php8.3 /var/www/moodle/admin/tool/analytics/cli/evaluate_model.php --modelid=1
```

## 12.3 Özel SQL Raporları (Configurable Reports Eklentisi)

```text
Site Administration > Plugins > Add-ons > Configurable Reports
```

Örnek SQL sorgular:

```sql
-- Son 30 günde giriş yapmayan aktif kullanıcılar (PostgreSQL)
SELECT u.firstname, u.lastname, u.email,
       TO_TIMESTAMP(u.lastlogin) AS son_giris
FROM mdl_user u
WHERE u.deleted = 0
  AND u.suspended = 0
  AND u.lastlogin > 0
  AND u.lastlogin < EXTRACT(EPOCH FROM NOW() - INTERVAL '30 days')::INTEGER
ORDER BY u.lastlogin ASC;
```

```sql
-- Ders bazında tamamlanma oranları
SELECT c.shortname, c.fullname,
       COUNT(DISTINCT cc.userid) AS tamamlayan,
       COUNT(DISTINCT ue.userid) AS toplam_kayitli,
       ROUND(COUNT(DISTINCT cc.userid)::numeric / NULLIF(COUNT(DISTINCT ue.userid), 0) * 100, 1) AS tamamlanma_oran
FROM mdl_course c
JOIN mdl_enrol e ON e.courseid = c.id
JOIN mdl_user_enrolments ue ON ue.enrolid = e.id
LEFT JOIN mdl_course_completions cc ON cc.courseid = c.id AND cc.userid = ue.userid AND cc.timecompleted IS NOT NULL
WHERE c.id > 1
GROUP BY c.id
ORDER BY tamamlanma_oran DESC;
```

> [!IMPORTANT]
> Moodle DB sorgularında tablo ön eki (`mdl_`) her zaman `{tablo_adi}` formatı yerine doğrudan yazılır. Sorgularınızı **Configurable Reports** eklentisine girerken `%%WWWROOT%%` ve `%%USERID%%` gibi dinamik değişkenler de kullanabilirsiniz.

## 12.4 Dashboard Yönetimi

```text
Site Administration > Appearance > Default Dashboard page
```

Öğrenci ve eğitmenlerin Dashboard'larını zorunlu bloklarla donatın:

| Blok | Hedef Kitle | Faydası |
| :--- | :--- | :--- |
| My Courses | Öğrenci | Kayıtlı dersler |
| Upcoming Events | Öğrenci/Eğitmen | Takvimdeki ödevler |
| Recent Activity | Öğrenci | Son forum yazıları |
| Course Overview | Eğitmen | Kurs durumu özeti |

---

> [!TIP]
> Learning Analytics modelleri çok fazla CPU tüketir. Büyük sitelerde bu modellerin hafta sonları veya gece saatlerinde çalışması için `Site Administration > Analytics > Model settings > Schedule` ayarını düzenleyin.
