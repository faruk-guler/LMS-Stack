# Bölüm 6: Kayıt Yönetimi (Recording Management)

BigBlueButton, toplantıdaki tüm ses, kamera, ekran paylaşımı, sohbet, beyaz tahta çizimleri ve sunum slaytı değişikliklerini ayrı ayrı kaydeder ve sonra bunları kronolojik sıraya dizerek (HTML5 "presentation" formatında veya saf Video formatında) sunar.

Kayıt süreci (Recording Pipeline) Ruby betiklerinden (scripts) oluşur ve oldukça CPU yoğun bir işlemdir.

## 6.1 Kayıt Yaşam Döngüsü

Bir toplantı sona erdiğinde (LMS tarafından sonlandırıldığında, odadan herkes çıkıp API otomatik kapattığında veya sunucu yeniden başlatıldığında) BBB sırasıyla 4 işlem (evreyi) uygular:

1. **Archive (Arşivleme):** `/var/bigbluebutton/recording/raw/` klasörüne, o odadaki anlık (ham) tüm Redis olayları, WAV dosyaları ve kamera videoları klasörlenir.
2. **Sanity (Bütünlük Kontrolü):** Ham veride bozukluk (ses ya da video yoksa) olup olmadığı taranır. Bozuksa süreç durabilir.
3. **Process (İşleme):** `/var/bigbluebutton/recording/process/presentation/` klasöründe ses, video senkronize edilir, sunum sayfaları PNG'lere çevrilir, tahta olayları eşlenir. En çok CPU yiyen evredir!
4. **Publish (Yayınlama):** `/var/bigbluebutton/published/presentation/` veya `/video/` klasörlerine HTML ve WebM dosyaları olarak aktarılır ve API'ye "Bu kayıt izlenmeye hazır" mesajı (Callback url) gönderilir.

> [!NOTE]
> İşleme (`Process`) evresi, 60 dakikalık bir ders için, sunucu gücüne bağlı olarak 15 dakika ila 1 saat sürebilir. Yoğun zamanlarda arkada bekleyen işler (Queued Tasks) birikebilir.

> [!IMPORTANT]
> **BBB 4.0 / LiveKit oturumlarında kayıt:** Varsayılan medya LiveKit olduğundan, kamera/ekran akışları **`bbb-webrtc-recorder`** servisi tarafından yakalanır (`recordingAdapter: bbb-webrtc-recorder`). Aşağıdaki `bbb-record` Ruby pipeline'ı ses/sunum/olay işleme için yine geçerlidir; ek olarak `bbb-presentation-video` paketi `events.xml` dosyasından MP4 render eder.

## 6.2 Kayıt Durumlarını ve Hatalarını Kontrol Etme

Sistemin arkasında işleyen Ruby süreçlerinin (record and playback) durumunu görmek için güçlü komut satırı aracı `bbb-record`'dur.

**Son dönemdeki kayıtların listesi:**

```bash
sudo bbb-record --list
```

**Kayıt işleyen işçi (worker) izlemek:**

```bash
sudo bbb-record --watch
```

> (Bu komut, anlık olarak hangi toplantının Archive, Process veya Publish evresinde olduğunu gösterir)

**Bir kaydı bilerek (manuel) yeniden işletmek (Rebuild):**

```bash
# Sadece spesifik bir Internal Meeting ID (Örn: 91b138e6...) için
sudo bbb-record --rebuild 91b138e6...
```

> Not: Bu id'yi `--list` komutunda görebilirsiniz.

## 6.3 Disk Kullanımı ve Eski Kayıtları Yönetme

BigBlueButton, sunucu disklerinin dolmasını önlemek için bir cron servisi kullanır (genellikle /etc/cron.d/bigbluebutton). Ancak bazen sysadmin'ler eski kayıtları elle temizlemek veya yedeklemek ister.

### Yayınlanmayan (Unpublished/Raw) Kayıtları Silme

Toplantı biteli onaylanmış ama API veya eğitimci tarafından "Kaydedilmeyecek" denmiş, silinmeyen ham veriler sunucuda kalabilir:

```bash
sudo bbb-record --deleteall
sudo bbb-record --clean
```

### Eski Yayınlanmış (Published) Kayıtları Silme Scripti

Otomatik olarak X günden eski **tüm kayıtları** silmek sunucuyu rahatlatır. BBB'nin içindeki şu dosyanın konfigürasyonunu (gün sayısını) düzenleyebilirsiniz:

```bash
nano /etc/cron.daily/bigbluebutton
```

Burada eski yayınları silen `history` değişkenini örneğin `-mtime +30` (30 günden eski) şeklinde ayarlayıp çalıştırabilirsiniz.

## 6.4 MP4 Olarak Kaydetmek Mümkün mü?

BigBlueButton varsayılan olarak "Presentation" (Sunum) kaydı formatı (.webm / html5 bazlı oynatıcı) verir. Eğer öğrenciler videoyu doğrudan `video.mp4` olarak indirmek isterse, "video" işleme formatı (video-playback) kurulmalıdır:

**(Kurulum ve aktifleştirme):**

```bash
sudo apt install bbb-playback-video
```

Bu paketi kurarsanız, kayıtlar iki farklı formatta işlenir (hem sunum, hem saf MP4 benzeri video). Ancak CPU işleme süresi **2 katına** çıkar.

## 6.5 Kayıtların Ortak Depolamaya (NFS vb.) Taşınması (Ölçeklendirme Ön Bilgisi)

Birden çok BBB sunucusu kuruluysa (Scalelite), kayıt işleme bitene kadar yerel diskte tutulur. Publish olduklarında otomatik NFS'ye yani `/mnt/scalelite-recordings/` tarzı ortak bir paylaşıma taşınmaları (Transfer) sağlanır. Buna 8. Bölüm'de (Scalelite) detaylıca değinilecektir.

> [!WARNING]
> **NFS İzin Krizleri (Error: Presentation Not Found):** Kayıt (mp4) dosyalarının ortak NFS diskine başarıyla yazılıp diğer sunuculardan da okunabilmesi için Linux'ta NFS alanını sisteme bağlarken (mount ederken) `uid=1000,gid=1000` (Yani Bigbluebutton özel kullanıcısı) parametreleriyle tanımlamalısınız. Dosya sahipliği `root:root` olarak geçerse eski kayıtlar izlenemez hale gelir.
