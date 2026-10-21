# Bölüm 4: Yönetim ve CLI Araçları

BigBlueButton sistem yöneticilerinin en yakın arkadaşı `bbb-conf` komut satırı aracıdır. Tüm servisleri koordine eden, hataları tespit eden ve log okumalarını kolaylaştıran bu betik, BBB'nin bel kemiğidir.

## 4.1 En Çok Kullanılan bbb-conf Komutları

### 1. `bbb-conf --status`

Tüm temel bileşenlerin ve servislerin o anki çalışma durumunu (Active/Inactive) listeler. BBB sürümünü en üstte gösterir. Nginx mi çöktü, Redis mi kapalı hızlıca teşhis etmek için kullanılır.

### 2. `bbb-conf --check`

Sadece servislerin ayakta olup olmadığına bakmaz, aynı zamanda yapılandırma dosyalarındaki mantıksal hataları, IP uyuşmazlıklarını ve sertifika sorunlarını tarar.

- Eğer "Potential problems" (Potansiyel Hatalar) başlığı altında uyarılar çıkarsa mutlaka düzeltilmelidir (Örneğin: Nginx 443'e bakıyor ama TLS sertifikası yok).

### 3. `bbb-conf --restart`

Bütün BigBlueButton servislerini (LiveKit, Mediasoup, bbb-html5, MongoDB, FreeSWITCH, bbb-web, GraphQL servisleri vb.) **doğru sırayla** durdurur ve yeniden başlatır.
> [!WARNING]
> Bu komut aktif tüm toplantıları anında keser ve herkesi sistemden atar. Kesinti yapılacağı zaman, gece saatlerinde veya kimsenin olmadığı zamanlarda kullanılmalıdır.

### 4. `bbb-conf --clean`

`--restart` işlemine benzer, ancak ek olarak temp (geçici) logları ve eski kalıntı verileri temizler (kayıtları silmez, sadece Redis vb. temp verilerini boşaltır). Sistemi "Taze" (Fresh) bir başlangıca zorlar. Beklenmedik "Oda açılamıyor" hatalarında hayat kurtarır.

### 5. `bbb-conf --stop` ve `bbb-conf --start`

Bütün BBB servislerini kapatır veya başlatır. Bakım (Backup/Restore veya OS Update) yapılacağı zaman önce `--stop` denilmelidir.

### 6. `bbb-conf --secret`

Moodle, Canvas LMS, WordPress eklentileri veya Greenlight ile API üzerinden konuşmak için gereken gizli anahtarı ve URL'i verir. Gerekirse sonuna `(new)` eklenerek oluşturulabilir (Örnek: `bbb-conf --setsecret yenisifrem`).

## 4.2 Önemli Servislerin Kontrolü (systemctl)

Bazen `bbb-conf` işe yaramaz veya tekil bir servise müdahale etmek istersiniz:

- **bbb-web:** `sudo systemctl restart bbb-web`
- **Frontend (NodeJS):** `sudo systemctl status bbb-html5`
- **GraphQL API (4.0):** `sudo systemctl status bbb-graphql-server bbb-graphql-middleware bbb-graphql-actions`
- **Medya (LiveKit — varsayılan):** `sudo systemctl status livekit-server`
- **Medya (Mediasoup — seçimlik):** `sudo systemctl status bbb-webrtc-sfu`
- **Veritabanı (Redis/Mongo):** `sudo systemctl status mongod`, `sudo systemctl status redis-server`
- **Ses (FreeSWITCH):** `sudo systemctl status freeswitch`

## 4.3 Log Dosyalarının Konumları ve İnceleme

BBB ile sorun yaşandığında bakılması gereken standart Linux log dizinleri vardır:

1. **Toplantı oluşturulamıyor / API hatası var:**

   ```bash
   tail -f /var/log/bigbluebutton/bbb-web.log
   ```

2. **Kameram açılmıyor, 1020 hatası alıyorum:**

   ```bash
   tail -f /var/log/bbb-webrtc-sfu/bbb-webrtc-sfu.log   # Mediasoup kullanılıyorsa
   journalctl -u livekit-server -f                        # LiveKit (varsayılan) kullanılıyorsa
   ```

3. **Kelimelerim/Sunumum tahtaya geç düşüyor veya kilitleniyor (Frontend hatası):**

   ```bash
   tail -f /var/log/bbb-html5/bbb-html5.log
   ```

4. **Nginx 404/502 Bad Gateway hatası veriyor:**

   ```bash
   tail -f /var/log/nginx/error.log
   ```

5. **Ses ile ilgili, mikrofona izin verince "1007/1004" Ses hatası:**

   ```bash
   tail -f /opt/freeswitch/var/log/freeswitch/freeswitch.log
   ```

> [!TIP]
> **`bbb-conf --watch` komutu:**
> Bu komutu çalıştırdığınızda tüm BBB bileşenlerinin son hata loglarını canlı olarak ekrana yazdırır (genel bir `tail` görünümü). Parametre olarak servis adı kabul etmez; tek servis izlemek için `sudo journalctl -fu bbb-web` gibi systemd komutlarını kullanın.

## 4.4 Apply-Config Scriptleri

BigBlueButton güncellendiğinde (`apt upgrade`) sizin `/etc/bigbluebutton/` gibi yerlerde yaptığınız bazı değişiklikler varsayılana dönebilir.

Bunu engellemek için `/etc/bigbluebutton/bbb-conf/apply-config.sh` dosyası oluşturulabilir. `bbb-conf --restart` komutu verildiğinde bu dosya varsa çalıştırılır ve örneğin sizin logonuzu (`favicon.ico`) otomatik olarak yerine geri koyar. Veya özel nginx ayarlarınızı enjekte eder.
