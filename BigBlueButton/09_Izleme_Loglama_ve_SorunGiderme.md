# Bölüm 9: İzleme (Monitoring), Loglama ve Sorun Giderme

BBB gibi çoklu bağımsız servisten oluşan platformlarda, sorun olmadan önce darboğazı tespit edebilmek için izleme şarttır. Ayrıca klasik WebRTC (Ses/Video) hatalarını çözebilmek bir SysAdmin için kritik yetenektir.

## 9.1 Prometheus ve Grafana Entegrasyonu

BigBlueButton topluluğu, metrikleri dışa aktaran `bigbluebutton-exporter` aracını geliştirmiştir.

> [!NOTE]
> Bu araç, BBB çekirdek ekibi tarafından değil, topluluk tarafından geliştirilmektedir. Güncel sürümü ve kurulum talimatları için aşağıdaki depoyu kontrol edin.

**Kurulum (BBB Sunucusunda):**

```bash
# Güncel kurulum talimatları için resmi repoya bakın:
# https://github.com/bigbluebutton/bigbluebutton-exporter

# Örnek kurulum (sürüm için repo'yu kontrol edin):
wget -qO- https://raw.githubusercontent.com/bigbluebutton/bigbluebutton-exporter/master/setup.sh | bash
```

Bu script, port `9688` üzerinden bağlı kullanıcı sayısı, çöken toplantı sayısı, CPU kullanımı, ses/video paket kayıpları gibi metrikleri (Prometheus formatında) dışarı verir.

Bir başka monitoring sunucusunda (Grafana yüklü olan), Prometheus hedeflerine bu IP'yi eklersiniz. Ve Grafana üzerinden hazır bir BBB Dashboard'u (Grafana.com üzerinde "BigBlueButton" aratarak güncel dashboard ID'sini bulun) ithal ederek anlık durumu grafiksel görebilirsiniz.

> [!TIP]
> **Log Takibi:** Sadece metrikleri değil, tüm servis loglarını (Nginx, Mediasoup, FreeSWITCH) merkezi olarak toplamak için Merkezi Log Yönetimi (`Integrations/03_Merkezi_Log_Yonetimi.md`) bölümüne göz atın.

## 9.2 Klasik Kullanıcı Hataları (WebRTC Error Codes)

Bağlanan öğrencilerin veya kullanıcıların ekranında sıklıkla bir "Hata Kodu" (Error 1020, 1007) belirebilir.

### Hata 1004 / 1007: Ses Bağlantısı Kurulamadı (ICE negotiation failed)

**Kullanıcının Şikayeti:** "Mikrofona izin veriyorum, Eko testine gelmeden bağlantı düştü hatası veriyor."
**Sebep ve Çözüm:**

- Kullanıcının okul/iş yeri güvenlik duvarı UDP paketlerini geçirmiyor.
- **Çözüm:** Sisteme düzgün bir TURN sunucusu kurulmamış veya TURN anahtarı (secret) hatalı girilmiş. TURN kuruluysa port kontrolü yapın.

### Hata 1020: Medya (Kamera/Ekran) Paylaşılamadı

**Kullanıcının Şikayeti:** "Kamera açıyorum, yükleniyor diyor siyah ekran kalıp kapanıyor."
**Sebep ve Çözüm:**

- Medya katmanının (varsayılan **LiveKit**; seçimlik Mediasoup) UDP portlarına (16384 - 32768) dışarıdan erişilemiyor.
- Sunucu NAT arkasındayken `dummy0` IP kartı yapılandırılmamış veya sistem `external_ip` değişkenini yanlış algılamış.
- **SysAdmin Müdahalesi:** `/opt/freeswitch/conf/sip_profiles/external.xml` (ses) ve medya SFU konfigürasyonu (`/etc/bigbluebutton/bbb-webrtc-sfu/production.yml` veya LiveKit `announcedIp`) içinde IP kontrolü yapın.

### 404 Not Found (Oda Açılmıyor / Katılınmıyor)

**Sebep ve Çözüm:**

- API (`bbb-web`) çökmüş veya Nginx API çağrılarını yönlendiremiyor.
- **SysAdmin Müdahalesi:** `sudo bbb-conf --status` yapıp `bbb-web` durumunu kontrol edin. `sudo systemctl restart bbb-web` deneyin.

## 9.3 Sistemin Kilitlenmesi ve Şişmesi

Eğer bir kullanıcı bbb-html5'in yavaşladığından, chat'in 5 saniye geç gittiğinden veya tahtaya çizilenin görünmediğinden bahsediyorsa sistem "NodeJS / CPU darboğazına" girmiş demektir.

1. **`top` ve `htop` Komutu ile Bütünleşik İnceleme:**
Eğer `mongod` veya `npm` (bbb-html5) process'leri sürekli %150-200 CPU kullanıyorsa, çok fazla tahta çizimi yapılmış veya gereksiz yere binlerce chat mesajı atılmış olabilir.
2. **Sorunlu Toplantıyı Kapatma:**
`bbb-web` üzerinden çok kaynak yiyen spesifik odanın internalID'sini bulup o odayı zorla kapatabilirsiniz (API aracılığıyla veya `bbb-conf --clean` diyerek herkesi düşürüp radikal bir restart ile).

## 9.4 "Potential PROBLEMS" Okuma Sanatı

`bbb-conf --check` yazıldığında çıkan warningleri doğru anlamak önemlidir.

Örnek Uyarı:

```text
Warning: The API secret does not match in bbb-web and bbb-apps-akka
```

**Anlamı:** Eski kurulumdan parçalar kalmış, BBB ana API şifresiyle Akka köprüsü uyuşmuyor demektir. (Birisi `--setsecret` komutu ile sırrı ezmiş). Çözüm: `bbb-conf --restart` yaparak cfg'lerin eşitlenmesini sağlamaktır.
