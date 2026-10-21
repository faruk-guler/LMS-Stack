# 🎓 LMS Stack — Moodle + BigBlueButton · Kurumsal Altyapı & Mimari (2026)

<p align="center">
  <strong>Moodle 5.3 (LTS)</strong> · <strong>BigBlueButton 4.0</strong> · <strong>Greenlight 3.x</strong> · <strong>PILOS 4.x</strong><br>
  Ekim 2026 · Türkçe · Açık Kaynak<br>
  <sub>Yazar: faruk-guler · www.farukguler.com</sub>
</p>

---

## 📌 Bu Dokümantasyon Nedir?

Bu set, en temelde **modüler bir LMS Stack**'i anlatır: bir öğrenme yönetim sistemi etrafında kurulan, birbirinin yerini alabilen **bileşen katmanlarının** uçtan uca Türkçe teknik dokümantasyonu. İki ana ürün ve ortak altyapı servisleri tek çatı altında toplanır — her klasör kendi modülünü, kesişen servisler ise kendi rehberini taşır. Kurumsal seviyede kurmak, yönetmek ve ölçeklemek isteyen sistem ve DevOps ekipleri için hazırlanmıştır; "next-next" rehberlerden farklı olarak gerçek dünya senaryolarına (sınav haftası darboğazları, NAT arkası sorunları, SSL krizleri, yüksek erişilebilirlik) odaklanır.

- **🟩 Moodle Nedir?** Dünyanın en popüler açık kaynaklı **öğrenme yönetim sistemi (LMS)**. PHP tabanlı; sınav, ödev, işbirliği ve analitik için kapsamlı ekosistem.
- **🟦 BigBlueButton Nedir?** Uzaktan eğitime odaklı açık kaynaklı **web konferans** sistemi. Beyaz tahta, breakout odaları, anket ve ekran paylaşımını doğrudan tarayıcıdan (HTML5) sunar; bu set güncel **v4.0** hattını esas alır.
- **🔗 Nasıl birleşir?** `mod_bigbluebuttonbn` / LTI köprüsü — Moodle kimlik–ders–katılımı yönetir, canlı sınıf oturumlarını BBB (+ ölçekleme için Scalelite) üstlenir.

### LMS Stack katmanları
| Katman | Bileşen(ler) | Rehber |
| :--- | :--- | :--- |
| Uygulama (LMS) | Moodle | `Moodle/` |
| Canlı sınıf | BigBlueButton · Greenlight/PILOS | `BigBlueButton/` |
| Kimlik / SSO | Keycloak · LDAP/SAML/OIDC | `Integrations/01` |
| Ölçekleme & HA | Scalelite · Sentinel · Patroni · keepalived | `Integrations/04`–`05` |
| Ortak servisler | TLS · Redis · Güvenlik | `Certs/` · `Redis/` · `Security/` |
| Gözlemlenebilirlik & Yedek | log/izleme · backup/DR | `Integrations/03` · `Backups/` |

---

## 📂 Proje Yapısı

```text
Moodle/          15 bölüm — Moodle 5.3 SysAdmin Rehberi
BigBlueButton/   17 bölüm — BBB 4.0 SysAdmin Rehberi
Integrations/     5 bölüm — SSO · e-posta/DMARC · log · Scalelite · HA & Failover
Certs/            Sertifika Rehberi (LE · CA/Sectigo · self-signed) + csr.conf
Redis/            Redis Rehberi (kurulum · güvenlik · maxmemory · Sentinel)
Security/         Güvenlik Rehberi (UFW · fail2ban · sysctl · SSH · izinler · WAF)
Backups/          Yedekleme & DR Rehberi (Golden Trio · betikler · PITR · restore)
Images/           görseller
Readme.md         tek README (bu dosya)
```

---

## 🛠️ Teknoloji Yığını

| Katman | Moodle | BigBlueButton |
| :--- | :--- | :--- |
| İşletim Sistemi | Ubuntu 22.04 / 24.04 | Ubuntu 24.04 + Docker |
| Uygulama | PHP 8.3 (FPM) | Java 21 (Grails 8) + Node.js 24 |
| Veritabanı | PostgreSQL 17 | PostgreSQL + MongoDB |
| Cache / MQ | Redis 7.x | Redis |
| Medya | — | LiveKit (varsayılan) + FreeSWITCH |
| Web / TLS | Nginx · Let's Encrypt | Nginx · Let's Encrypt |

---

## 🎯 Kimler İçin?

Kurumsal LMS + video konferans altyapısını **kurmak, ölçeklemek, izlemek** ve kriz anında (502 Gateway, WebRTC 1007/1020, "beyaz ekran") çözüm üretmek isteyen sistem/DevOps ekipleri ile üniversite ve kurumsal eğitim yöneticileri.

---

## 📋 Hızlı Başlangıç

```bash
# Moodle — sürüm, cache temizle, cron durumu
cat /var/www/moodle/version.php | grep "\$release"
sudo -u www-data php8.3 /var/www/moodle/admin/cli/purge_caches.php
sudo journalctl -u moodle-cron.service --since "1 hour ago"

# BigBlueButton — sürüm / servis durumu / kontrol + API secret
sudo bbb-conf --version && sudo bbb-conf --status && sudo bbb-conf --check
sudo bbb-conf --secret
```

---

## 📄 Lisans

Bu dokümantasyon seti açık kaynak olarak paylaşılmaktadır; eğitim amaçlı kullanılabilir, dağıtılabilir ve katkıda bulunulabilir.

<p align="center"><sub>⭐ Beğendiyseniz yıldız bırakmayı unutmayın — faruk-guler</sub></p>
