# Bölüm 3: Ağ, Güvenlik Duvarı ve TURN/STUN Yapılandırması

BigBlueButton'ın en hassas kısmı network (ağ) yapılandırmasıdır. Kullanıcıların ses veya kamerasının bağlanamaması (meşhur 1007 veya 1020 WebRTC hataları) genellikle ağ veya NAT arkasındaki yapılandırma eksikliklerinden kaynaklanır.

## 3.1 Güvenlik Duvarı (Firewall) Port İzinleri

Bir ortamdaki güvenlik duvarı (ister UFW, ister harici bir donanım firewall - FortiGate, pfSense vb.) BBB sunucusuna aşağıdaki portlara erişim izni vermelidir:

| Port | Protokol | Durum (In/Out) | Görev |
| --- | --- | --- | --- |
| 80/443 | TCP | In/Out | HTTPS (Web) arayüzü ve API çağrıları |
| 7443 | TCP | In/Out | Güvenli WebSocket / FreeSWITCH |
| 16384 - 32768 | UDP | In/Out | Mediasoup / WebRTC (Kamera, Ses RTP / RTCP paketleri) |
| 3478 | UDP/TCP | In/Out | TURN/STUN (Coturn) |
| 5060 | UDP | **Kısıtlı** | FreeSWITCH SIP — Sadece SIP Sağlayıcısı IP'sinden izin verin |

> [!WARNING]
> **SIP Portu (UDP 5060)** internete açık bırakıldığında günde binlerce korsan arama (Toll Fraud) girişimiyle karşılaşabilirsiniz. UFW üzerinde UDP 5060'ı **yalnızca** SIP sağlayıcınızın IP aralığına kısıtlayın. SIP Trunk entegrasyonu kullanmıyorsanız bu portu tamamen kapalı tutun.

## 3.2 İç Ağ ve NAT Arkasına Kurulum (Dummy NIC Çözümü)

Kurumsal firmalarda veya ev ağlarında sunucular bazen doğrudan dışarıya bakan statik bir "Public IP" (Dış İP) almazlar. İçeride örneğin `192.168.1.100` adresine sahiptirler ve modem/firewall üzerinden Public IP (örn: `203.0.113.1`) NAT'lanır (Port Forwarding).

BBB kurulumu normal şartlarda sisteminde "Public IP" bulamazsa FreeSWITCH ve WebRTC doğru IP'yi istemcilere anons edemez, bu da "Dışarıdaki insanlar benim sesimi duyamıyor" sorununa (1020 WebRTC hatası) yol açar.

Bu sorunu çözmenin yöntemleri:

### Yöntem 1: Dummy Interface (Sahte Ağ Arabirimi)

Sunucu içine dış IP'mizmiş gibi davranacak sahte bir ağ kartı (Interface) eklemek.

1. `/etc/netplan/` içindeki yapılandırma dosyanıza (ör. `00-installer-config.yaml` veya `50-cloud-init.yaml`) şunu ekleyin:

```yaml
network:
  version: 2
  ethernets:
    ens160: # Orijinal ağ kartınız (örnek ismidir)
      dhcp4: true
    dummy0: # Yeni Eklenen
      match:
        name: dummy0
      addresses:
        - 203.0.113.1/32 # Sizin gerçek Public (Dış) IP'niz
```

1. Sisteme "dummy" modülünü yükleyin:

```bash
sudo modprobe dummy
echo "dummy" | sudo tee -a /etc/modules
```

1. Netplan'ı uygulayın:

```bash
sudo netplan apply
```

*Artık `ifconfig` yazdığınızda sistemin `dummy0` arayüzünde `203.0.113.1` (dış IP) gördüğünü anlayacak ve FreeSWITCH bu adresi anons edecektir.*

### Yöntem 2: FreeSWITCH SIP profiline Extern-IP Eklemek

(Tavsiye edilen dummy'dir, ama zorunlu kalırsanız) `sip_profiles/external.xml` tarzı dosyalardaki `$${local_ip_v4}` değişkenlerini `$${external_rtp_ip}` formatında zorla belirtmek gerekir. (Bu konu `bbb-conf --setip` ile bazen ezildiği için risklidir).

## 3.3 TURN ve STUN (Coturn) Nedir? Neden Gerekir?

### STUN

Katılımcıya "Senin internette görünen Public IP adresin bu" diyen bir ayna servisidir.

- Eğer istemci (öğrenci/katılımcı) evinde simetrik NAT/Firewall arkasındaysa sadece STUN işe yaramaz.

### TURN (Traversal Using Relays around NAT)

Katılımcının (veya sunucunun) ağ kısıtlamaları (firewall) veya asimetrik NAT yüzünden **doğrudan** UDP (16384-32768) iletişimini kuramadığı durumlarda aracı bir Tünel gibi davranır.

- İşyeri ağından, katı kısıtlamalı otel ve okul ağlarından bağlanan kullanıcılar TURN sunucusu üzerinden medyayı (443 portundan şifreli olarak) tüneller.
- Bu sayede 1020 (Medya Bağlantısı Kurulamadı) hataları engellenir.

### Coturn Kurulumu (BBB'den Ayrı Sunucu Önerilir!)

> 🔑 Coturn için sertifika (üretim + `/etc/turnserver.conf`'a kurulum; `turn.sirket.com` SAN'lı **ayrı** cert) dahil **tüm sertifika işlemlerinin tek kaynağı**: Certs — Sertifika Kurulum Rehberi (`Certs/Sertifika_Rehberi.md`).

> [!TIP]
> BBB'yi kurduğunuz sunucuya aynı zamanda TURN kurabilirsiniz, fakat yoğunluk arttığında ağ darboğazı olabilir. Tavsiye edilen, `turn.sirket.com` gibi ucuz bir sanal makine (Örn. 2 CPU, 4 GB RAM ubuntu) yaratıp sadece Coturn kurmaktır.

**Ayrı Sunucuda Ubuntu 24.04'e Coturn Kurulumu (Port 443/TCP Tavsiyesi):**

> [!IMPORTANT]
> `bbb-install.sh`'in **`-c` bayrağının biçimi `<hostname>:<secret>`tir** — ikinci alan **port değil, senin seçtiğin 8–16 karakterlik shared secret**'tır. (`-p` bayrağı Coturn portu değil, **apt-get proxy**'sidir; karıştırma.) Coturn'ün 443/TCP'te dinlemesi `-c` ile değil, coturn'un kendi `turnserver.conf` dosyasında (`listening-port=3478`, `tls-listening-port=443`) ayarlanır.

1. Sunucuyu hazırlayın. Kurumsal firewall'ları (Deep Packet Inspection olmayanları) baypas etmek için Coturn'ü `turnserver.conf` üzerinden standart **443/TCP** portunda dinleyecek şekilde yapılandırın.
2. Secret'ı siz üretin (örn. `openssl rand -hex 8`), ardından coturn'u **`-v` olmadan** kurun (`-e` ile Let's Encrypt cert'i otomatik alınır):

```bash
wget -qO- https://raw.githubusercontent.com/bigbluebutton/bbb-install/refs/heads/v4.0.x-release/bbb-install.sh | bash -s -- \
  -c turn.sirket.com:GIZLI_ANAHTAR -e admin@sirket.com
```

> [!IMPORTANT]
> TURN sunucusunda 443/TCP kullanacaksanız, o sunucuda Nginx veya başka bir web sunucusunun 443 portunu **işgal etmediğinden** emin olun.

**BBB Sunucunuzu TURN'e Bağlamak:**

BBB'yi harici coturn'e yönlendirmek için BBB kurulum komutunu **`-v` ile tekrar çalıştırıp** sonuna aynı `-c <hostname>:<secret>` değerini ekleyin (resmî `bbb-conf` araç setinde `--setturn` adında bir bayrak **yoktur**):

```bash
wget -qO- https://raw.githubusercontent.com/bigbluebutton/bbb-install/refs/heads/v4.0.x-release/bbb-install.sh | bash -s -- \
  -v noble-400 -s bbb.sirket.com -e admin@sirket.com -c turn.sirket.com:GIZLI_ANAHTAR
```

Bu adım, `bigbluebutton.properties` içindeki `turnServer`/`turnSecret` değerlerini `-c` ile verdiğin hostname:secret ile senkronize eder.

1. `sudo bbb-conf --check` yazarak `turn.sirket.com` ile düzgün iletişim kurulduğunu kontrol edin.

## 3.4 Çıplak IP Konusu ve IPv6

BigBlueButton kurulumlarında her zaman domain (FQDN) kullanılmalıdır. IP adresi ile SSL sertifikasını geçersiz kılacak şekilde (self-signed) kurabilirsiniz ancak tarayıcılar (Chrome/Firefox vb) "Kameralara ve Mikrofona erişim" iznini güvenli olmayan (HTTP veya geçersiz HTTPS) sitelerde vermeyecektir.
Bazı operatörlerde IPv6 ile `bbb-web` arasında çökmeler olabilir. Sorun yaşarsanız IPv6'yı sunucu bazında devre dışı bırakmayı deneyebilirsiniz:

```bash
sysctl -w net.ipv6.conf.all.disable_ipv6=1
```
