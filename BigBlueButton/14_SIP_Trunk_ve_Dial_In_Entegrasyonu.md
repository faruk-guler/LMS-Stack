# Bölüm 14: FreeSWITCH SIP Trunk ve Dial-In Entegrasyonu

BigBlueButton, sadece tarayıcı üzerinden web kameralı ve mikrofonlu katılımı (WebRTC) desteklemez. Altyapısındaki **FreeSWITCH** bileşeni, gerçek bir telefon santrali (VoIP Softswitch) gibi çalışabilir.

## 14.1 Dial-In (Telefonla Katılım) Nedir?

Dial-in sistemi ile katılımcılar, internetleri olmadığında ya da bilgisayar başında olmadıklarında, sabit bir telefon numarasını (Örn: `+90 212 555 12 34`) cep telefonlarından arayarak odaya sadece "Sesli" katılabilirler.

Bunun çalışabilmesi için bir Telekom sağlayıcısından (TTNet, TürkNet, Twilio, Plivo vb.) bir SIP Trunk (Sanal Santral Hattı) kiralamanız gerekmektedir.

## 14.2 FreeSWITCH'e SIP Trunk Eklemek

BigBlueButton içindeki FreeSWITCH, kendi kendine gelen aramaları dinleyebilir. Bunun için dış sağlayıcınızla aranızda bir köprü (Gateway) kurmalısınız.

`/opt/freeswitch/conf/sip_profiles/external/tutkun_telekom.xml` (isim örnek) adında bir dosya oluşturun:

```xml
<include>
  <gateway name="tutkuntelekom">
    <!-- Sağlayıcınızın verdiği bilgiler -->
    <param name="username" value="SIP_KULLANICI_ADINIZ"/>
    <param name="password" value="SIP_SIFRENIZ"/>
    <param name="realm" value="sip.tutkuntelekom.com.tr"/>
    <param name="proxy" value="sip.tutkuntelekom.com.tr"/>
    
    <!-- Gelen aramaları içeri almak için kayıt (register) -->
    <param name="register" value="true"/>
  </gateway>
</include>
```

Kaydettikten sonra FreeSWITCH'e bu yeni dış köprüyü yüklemesini ('reload') söylemeniz gerekir (veya kolay yoldan `systemctl restart freeswitch`).

## 14.3 Gelen Aramayı (Inbound Route) Toplantıya Yönlendirmek (Dialplan)

Sabit hattınız (Örn: 0212 555 1234) çaldığında, FreeSWITCH'in telefonu açıp *"Lütfen konferans pin kodunu giriniz"* demesi lazımdır.

Bunun için `/opt/freeswitch/conf/dialplan/public/` altına bir XML kuralı yazılır (Örneğin `00_inbound_pstn.xml`):

```xml
<include>
  <extension name="Gelen_Telekom_Aramalari">
    <!-- Sağlayıcıdan gelen DID yani telefon numaranız -->
    <condition field="destination_number" expression="^902125551234$">
      <!-- Aramayı aç -->
      <action application="answer"/>
      
      <!-- BigBlueButton'ın varsayılan "Pin Soran" karşılama senaryosuna gönder -->
      <action application="sleep" data="500"/>
      <action application="play_and_get_digits" data="5 5 3 7000 # conference/conf-pin.wav ivr/ivr-that_was_an_invalid_entry.wav pin \d+"/>
      <action application="transfer" data="${pin} XML default"/>
    </condition>
  </extension>
</include>
```

## 14.4 API Üzerinden Konferans Pini Oluşturmak (VoiceBridge / DialNumber)

Telefonla katılacak kişilerin tuşlayacağı (Pin Numarası), aslında BBB API'sinin `voiceBridge` parametresidir.

Öğretmen odayı başlattığında Greenlight ekranda bir mesaj gösterir: *"Telefonla katılmak isterseniz 02125551234'yi arayın ve PIN kodunu girin: 72834"*

Bu 5 haneli PIN (voiceBridge), arayan kişiyi doğru öğretmenin/dersin odasına sesli olarak bağlar. Tarayıcıdan bağlananlarla telefonla bağlananlar birbiriyle konuşabilir hale gelir.

> [!WARNING]
> SIP portu (UDP 5060), internete direkt açık olduğunda günde binlerce "Korsan Arama (Toll Fraud)" yiyecektir. UFW Güvenlik duvarından 5060 portuna SADECE SIP Sağlayıcınızın (Örn Telekom Firması) IP havuzundan erişilmesine izin vermelisiniz.

## 14.5 Kullanıcıların Odadan Telefon Araması Yapması (Dial-Out) - Opsiyonel İleri Seviye

Eğer bir katılımcı toplantıdayken bir GSM numarasını arayıp onu da odaya çekmek isterse, BBB'nin resmi GitHub sayfasındaki eklentilerden `bbb-webrtc-sfu` ve FreeSWITCH modifikasyonları ile giden arama (Dial-Out) da etkinleştirilebilir (Çok nadir bir senaryodur ve yüksek güvenlik riski - yüksek fatura riski taşır).
