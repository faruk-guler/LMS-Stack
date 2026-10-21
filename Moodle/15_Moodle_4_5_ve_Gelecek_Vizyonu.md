# 15 - Moodle 5.x Yenilikleri ve Gelecek Vizyonu (2026)

Eğitim dünyası değişiyor, Moodle da bu değişime sürüklenerek değil **liderlik ederek** ayak uyduruyor. 2026 ve ötesinde bizi neler bekliyor?

## 15.1 Moodle 5.x Serisi — Sürüm Tablosu

| Sürüm | Yayın Tarihi | Öne Çıkan Özellik |
| :--- | :--- | :--- |
| Moodle 4.5 (LTS) | Eki 2024 | Uzun destekli sürüm; raporlama ve kullanıcı yönetiminde iyileştirmeler |
| Moodle 5.0 | Nis 2025 | PHP 8.2 taban; çekirdek modernleştirmeleri |
| Moodle 5.1 | Eki 2025 | **`/public` dizin mimarisi**, Oracle desteği kaldırıldı |
| Moodle 5.2 | Nis 2026 | PHP 8.3 tabana yükseltildi; performans güncellemeleri |
| **Moodle 5.3 (LTS) — Güncel** | **Eki 2026** | **Kurumsal LTS hedefi: PHP 8.3, PostgreSQL 17, MariaDB 11.4 / MySQL 8.4** |

## 15.2 Moodle 5.3 LTS'e Geçiş Kontrol Listesi

5 Ekim 2026'da yayımlanan Moodle 5.3, önümüzdeki 3-5 yılın standart kurumsal sürümü (LTS) olacaktır. Sorunsuz geçiş için şu adımları tamamladığınızdan emin olun:

- [ ] **PHP 8.3 Geçişi:** Moodle 5.3 için taban **PHP 8.3**'tür (8.4 de desteklenir ve önerilir). Eski PHP 8.1/8.2 kurulumlarını emekli edin.
- [ ] **Veritabanı Güncelleme:** PostgreSQL **17** veya MySQL 8.4 / MariaDB 11.4 sürümlerine geçiş yapın. Eski `lower_case_table_names` veya `collations` ayarlarını Moodle standartlarına getirin.
- [ ] **`/public` Mimari Dönüşümü:** Web sunucunuzu (Nginx/Apache), kök dizin `/public` olacak şekilde yeniden yapılandırın (bu mimari **5.1** ile geldi). Gelecekteki güvenlik güncellemeleri bu mimariyi baz alacaktır.
- [ ] **SSO Modernizasyonu:** Greenlight v3 geçişi ile birlikte LDAP yerine OIDC (Keycloak) mimarisine geçişinizi tamamlayın.
- [ ] **UI/UX Sadeleştirme:** Eski "Legacy" temaları terk edin; Moodle 5.x'in modern AI-destekli arayüzüne uyumlu özel temalar geliştirin.

## 15.3 Yapay Zeka (AI) Entegrasyonu

Moodle **4.5** ile birlikte çekirdek seviyesinde AI altyapısı (AI providers / AI actions) devreye girdi.

```text
Site Administration > AI > AI providers
```

```php
// config.php — AI provider bağlantısı (isteğe bağlı)
// OpenAI veya Azure AI ayarları admin panelinden da yapılabilir
```

**Kullanım Senaryoları:**

- **Eğitmenler için:** Kurs içeriği taslağı oluşturma, soru üretimi
- **Öğrenciler için:** Ödev özeti, kavram açıklaması
- **Yöneticiler için:** Rapor yorumlama yardımı

> [!NOTE]
> AI özelliklerini açmak isteğe bağlıdır. Veri gizliliği politikanıza göre OpenAI gibi harici servislere veri gönderildiğini kullanıcılara bildirmeniz yasal açıdan önemlidir.

## 15.4 Modern PHP ve Performans Artışı

```bash
# Moodle 5.x için PHP 8.3 önerilir (taban 8.3)
php8.3 -v

# PHP 8.3 JIT Compiler (Just-In-Time) aktif etme
# php.ini
opcache.jit = 1255
opcache.jit_buffer_size = 64M
```

PHP 8.3 JIT ile birlikte CPU-yoğun Moodle işlemleri (Analytics modelleri, büyük raporlar) %20-40 daha hızlı çalışabilmektedir.

## 15.5 Moodle LMS — Docker ile Kurulum (Geliştirme/Test Ortamı)

> [!WARNING]
> Aşağıdaki Docker Compose sadece geliştirme ve test ortamları içindir. **Canlı (production) Moodle kurulumu için önerilmez.**

```yaml
# docker-compose.yml
version: '3.8'

services:
  moodle:
    image: bitnami/moodle:5
    ports:
      - "8080:8080"
      - "8443:8443"
    environment:
      - MOODLE_DATABASE_TYPE=pgsql
      - MOODLE_DATABASE_HOST=moodle-db
      - MOODLE_DATABASE_NAME=moodledb
      - MOODLE_DATABASE_USER=moodleuser
      - MOODLE_DATABASE_PASSWORD=sifre123
      - MOODLE_USERNAME=admin
      - MOODLE_PASSWORD=Admin1234!
      - MOODLE_EMAIL=admin@example.com
      - MOODLE_SITE_NAME=Test Moodle
    volumes:
      - moodle_data:/bitnami/moodle
      - moodle_data_dir:/bitnami/moodledata
    depends_on:
      - moodle-db

  moodle-db:
    image: postgres:17
    environment:
      - POSTGRES_DB=moodledb
      - POSTGRES_USER=moodleuser
      - POSTGRES_PASSWORD=sifre123
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  moodle_data:
  moodle_data_dir:
  postgres_data:
```

```bash
docker compose up -d
# Moodle'a http://localhost:8080 adresinden erişin
```

## 15.6 Moodle Roadmap — 2026 ve Ötesi

- **Stateless Moodle:** Kubernetes üzerinde yatay ölçeklenen Moodle küme mimarileri standart hale geliyor.
- **Micro-learning:** Daha kısa, odaklı öğrenme modülleri için optimizasyon.
- **SCORM 3.0 (xAPI/cmi5):** Yeni nesil SCORM standardına geçiş.
- **Moodle Workplace:** Kurumsal eğitim ve sertifikasyon yönetimi özellikleri Core'a taşınıyor.

---

> [!TIP]
> **Son Söz:** Moodle bir yazılım değil, yaşayan bir organizmadır. Onu güncel, güvenli ve performanslı tutmak sadece kodla değil, iyi bir sistem yönetimi vizyonuyla mümkündür. 2026 rehberinin sonuna geldik. İyi şanslar!
