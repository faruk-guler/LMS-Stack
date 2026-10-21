# Bölüm 1: BigBlueButton Mimarisi ve Temel Bileşenleri (2026 Standartları)

BigBlueButton (BBB), web tabanlı, açık kaynaklı bir video konferans sistemidir. Eğitim odaklı geliştirilmiş olup, ekran paylaşımı, ortak beyaz tahta (whiteboard), anketler, paylaşılan notlar gibi özellikler sunar. Bu rehber, güncel kararlı hat olan **v4.0**'ı esas alır; sağlam ve ölçeklenebilir bir yapı sunabilmek için çeşitli modern ve açık kaynaklı yazılımların bir araya gelmesiyle oluşur.

## 1.1 Genel Mimari

BigBlueButton'ın çekirdeği, farklı işlevleri yerine getiren bağımsız servislerin Nginx arkasında entegre çalışmasına dayanır. İstemci (tarayıcı) doğrudan Nginx ile HTTP/HTTPS ve WebSocket üzerinden konuşur.

> [!NOTE]
> Tüm medya (ses, video, ekran paylaşımı) WebRTC protokolü üzerinden aktarılır. Bu, istemcilerde herhangi bir eklenti (plugin) kurulumu gereksinimini ortadan kaldırır.

### İstek Akışı Özeti

1. İstemci `https://bbb.sirket.com` adresine Nginx üzerinden bağlanır.
2. Web uygulaması (Greenlight veya özel istemci) üzerinden toplantı başlatma isteği `bbb-web` API'sine iletilir.
3. Toplantı başlatıldığında istemci, HTML5 arayüzünü (bbb-html5) yükler.
4. Ses ve video akışları için WebSocket üzerinden sinyalleşme yapılır ve medya sunucularına bağlanılır. **Varsayılan medya katmanı LiveKit'tir** (ses, kamera ve ekran paylaşımı); Mediasoup (bbb-webrtc-sfu) ve FreeSWITCH seçimlik olarak kuruludur.

## 1.2 Temel Bileşenler (Core Components)

### 1. Nginx (Web Sunucusu ve Reverse Proxy)

Sistemin dışarıya açılan kapısıdır. Tüm HTTP/HTTPS isteklerini ve WebSocket sinyalleşme trafiğini alır, uygun servislere (bbb-web, bbb-html5, FreeSWITCH vb.) yönlendirir (reverse proxy). Statik dosyaları (HTML, CSS, JS) istemcilere sunar.

### 2. bbb-web (BigBlueButton API)

Java(Grails) tabanlı ana backend uygulamasıdır.

- Toplantı oluşturma, sonlandırma, kullanıcı katılımı gibi API isteklerini yönetir.
- Toplantı durumunu izler.
- Redis'e toplantı olaylarını göndererek `bbb-html5` backend'i ile haberleşir.

### 3. bbb-html5 (NodeJS/Meteor Uygulaması)

Kullanıcının tarayıcısında çalışan arayüzün (Frontend) sunucu tarafı yansımasıdır (Backend).

- İki ana parçadan oluşur: `Frontend` (tarayıcıda çalışan React kodu) ve `Backend` (NodeJS).
- Sohbet mesajları, beyaz tahta çizimleri, anket sonuçları gibi anlık durum verilerini yönetir.

### 4. Redis (In-Memory Veri Yapısı Mağazası)

BBB bileşenleri arasındaki "Mesajlaşma Veriyolu" (Message Bus) görevini üstlenir.

- `bbb-web`, `bbb-html5` ve `bbb-apps-akka` arasındaki iletişimi sağlar (Pub/Sub mimarisi üzerinden).
- Toplantıdaki geçici sohbet veya katılımcı durumlarını hafızada tutar.

> [!NOTE]
> BBB Redis'i **node-lokaldır ve kümeleştirilmez**; HA node düzeyinde Scalelite ile gelir. Redis kurulumu/güvenliği/`maxmemory-policy noeviction`/Sentinel ayrıntısı için tek kaynak: `Redis/Redis_Rehberi.md`.

### 5. MongoDB (Belge Tabanlı Veritabanı)

`bbb-html5` sunucu tarafı uygulaması, o an aktif olan toplantının tüm durumunu (kimler bağlı, tahtada ne çizilmiş, paylaşılan notlar vb.) MongoDB üzerinde depolar. Kalıcı veriden ziyade aktif oturumların durumunu tutmak için kullanılır (Ephemeral state).

### 6. FreeSWITCH (Ses ve Telefon Bileşeni)

- Softswitch / SIP sunucusudur.
- Kullanıcıların mikrofonlarından gelen sesi işler, miksler ve diğerlerine dağıtır.
- İstenirse harici SIP trunk'lar bağlanarak normal telefon hatlarından toplantıya katılım sağlanabilir (Dial-in).

### 7. Medya Katmanı: LiveKit (varsayılan) / Mediasoup (bbb-webrtc-sfu)

**SFU (Selective Forwarding Unit)** mimarisinde çalışarak kamera videolarını ve ekran paylaşımlarını yönetir.

- **LiveKit varsayılan medya çatısıdır** (ses, kamera ve ekran paylaşımı). `bbb-webrtc-sfu` (Mediasoup) ve FreeSWITCH kurulu kalır ve sunucu geneli ya da toplantı bazında seçilebilir.
- Kurento mimariden çıkarılmıştır. Kalabalık kameralı toplantılarda SFU yaklaşımı CPU yükünü ciddi oranda düşürür.

### 8. Kayıt Sistemi (Recording Services)

Ruby scriptlerinden oluşur. Toplantı bitip `bbb-web` "Toplantı bitti" sinyali gönderdiğinde çalışmaya başlar.

- **Archive:** Ham medya (ses, video, olaylar) diskte arşivlenir.
- **Sanity:** Ses ve video dosyalarının bütünlüğü kontrol edilir.
- **Process:** Medyalar birbiriyle senkronize edilerek işlenir (Genellikle MP4 veya WebM formatına dönüştürülür).
- **Publish:** İşlenen çıktı, web üzerinden izlenebilecek bir "Presentation" veya "Video" dizini altına publish edilir.

### 9. Greenlight (İsteğe Bağlı - Front-End Uygulaması)

BBB'nin kendisi sadece bir API sunar, kendi başına kullanıcıların giriş yapıp oda açacağı bir web arayüzü yoktur. Greenlight v3 (PostgreSQL & React tabanlı), BigBlueButton projesi tarafından sunulan varsayılan ve modern son kullanıcı arayüzüdür.

- Kullanıcı kaydı, şifre yönetimi, oda (Room) oluşturma ve paylaşma sağlar.

---

> [!TIP]
> **Özetle:** Bir kullanıcı BBB sunucusuna bağlandığında, Nginx port 443 üzerinden HTML sayfasını alır. Chat/Tahta gibi işlemler için NodeJS (`bbb-html5` + Redis + MongoDB) devreye girer. Kamerayı açtığında medya katmanı (varsayılan **LiveKit**) portları (UDP 16384-32768) üzerinden video gönderilir. Ses de aynı medya çatısı üzerinden (seçime göre FreeSWITCH) işlenir. Bütün bu orkestrasyonu `bbb-web` API'si yönetir.
