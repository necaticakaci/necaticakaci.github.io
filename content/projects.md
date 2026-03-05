---
title: "Projeler"
description: Bazı açık kaynaklı projelerim
disableComments: true
---

---

## Mihenk

<p></p>

Mihenk, gömülü sistemler için kolay taşınabilir ve standart kütüphane gerektirmeyen bir başarım ölçüm programıdır. Matris çarpımı ve AES şifreleme gibi yaygın kullanılan birçok algoritmayı işlemci üzerinde çalıştırıp nihai puan verir.

* [Mihenk GitHub](https://github.com/necaticakaci/mihenk)

## Matrak

<p><img src="/matrak-logo.png" width="250" height="150"/></p>

Matrak, Verilog ile yazılmış basit bir 32-bit RISC-V işlemcidir. Detaylı işlemci tasarım rehberi hazırlamak amacıyla geliştirilmiştir.

* [Matrak GitHub](https://github.com/necaticakaci/matrak)
* [RISC-V İşlemci Tasarımı Yazı Dizisi](/posts/riscv-1)

---

<details>
<summary>Arşivlenmiş projeleri göstermek için tıklayın</summary>

## Manta C211 (2021)

<img src="/manta-logo.png" width="250" height="150"/>

VHDL ile yazılmış Picoblaze benzeri 8-bit işlemci.

* [Manta C211 GitHub Repo](https://github.com/necaticakaci/manta-c211)
* [MANTASM](https://github.com/necaticakaci/manta-c211/blob/main/mantasm/mantasm.py): Manta C211 işlemci için assembler

---

## Nükleon Linux (2020)

<p></p>

<p><img src="/nukleon.png"/></p>

Nükleon Linux projesi, ISUBÜ Savunma Uzay ve Havacılık Topluluğu bünyesinde hazırladığımız Debian tabanlı bir Linux işletim sistemidir. İşletim sistemi, mühendislik çalışmaları için özelleştirilmiş birçok açık kaynaklı yazılım ile birlikte geliyordu.

<details>
<summary><mark>Orjinal readme dosyasını göstermek için tıklayın</mark></summary>

```
              __   .__                        .__  .__                     
  ____  __ __|  | _|  |   ____  ____   ____   |  | |__| ____  __ _____  ___
 /    \|  |  \  |/ /  | _/ __ \/  _ \ /    \  |  | |  |/    \|  |  \  \/  /
|   |  \  |  /    <|  |_\  ___/  |_| |   |  \ |  |_|  |   |  \  |  />    < 
|___|  /____/|__|_ \____/\___  \____/|___|  / |____/__|___|  /____//__/\_ \
     \/           \/         \/           \/               \/            \/
###########################################################################
Nükleon Linux 1.0 "MUAVENET" Beta Önizleme Sürümü
Nükleon Linux mühendislik ve bilimsel çalışmalar için özelleştirilmiş, 
Debian kararlı sürüm tabanlı açık kaynaklı bir Linux işletim sistemidir. 

###########################################################################
SIK SORULAN SORULAR

1.Nükleon Linux'u bilgisayarıma kurmadan deneyebilir miyim?
Elbette. Nükleon Linux çalışan kalıp olarak dağıtılır, dolayısıyla varolan 
işletim sisteminizde değişiklik yapmadan deneyebilirsiniz. Bunun yanı sıra
sanal makinelerde de test edebilirsiniz.

2.Nükleon Linux'ta daha önce kullandığım programları çalıştırabilir miyim?
Nükleon Linux, özelleştirilmiş bir Debian Linux işletim sistemidir. 
Synaptic paket yöneticisi veya apt komutu ile Debian deposundaki paketleri
rahatlıkla kurabilirsiniz. Ayrıca Gdebi programı ile de kendi .deb uzantılı
paketlerinizin kurulumlarını gerçekleştirebilmeniz mümkündür. 

Microsoft Windows üzerinde çalışan birçok programın Linux için uygun
sürümleri mevcuttur. Ne yazık ki programların bazılarının  Linux desteği 
yoktur. (Linux desteği olmayan bir programı çalıştırmayı denemek için wine
paketini yükleyebilirsiniz. Bkz. https://wiki.winehq.org/Debian)

3.Neden Nükleon Linux kullanmalıyım?
A. Nükleon Linux özgürdür ve ücretsiz olarak dağıtılır. 
B. Nükleon Linux modern ve kullanışlı Cinnamon masaüstü arayüzü ve önceden
yüklenmiş harika araçlar ile birlikte gelir. Yeni kullanıcılar, kullanıcı 
dostu arayüze kolaylıkla adapte olabilir.
C. İlk iki madde yeterince ikna edici değil ise sanal makine üzerinde
deneyip kararınızı verebilirsiniz. Bkz. https://www.virtualbox.org/

4.Nükleon Linux'u eski bilgisayarıma kurabilir miyim?
Tabi ki. Sistem gereksinimlerini karşılamakta olan her x64 mimari 
destekleyen bilgisayara kurabilirsiniz.

Önerilen Sistem Gereksinimleri:
Çift çekirdekli x64 işlemci
1 GB RAM
25 GB HDD

5.Nükleon Linux'un arkasında kim var?
Nükleon Linux, Isparta Uygulamalı Bilimler Üniversitesi, Savunma Uzay ve
Havacılık topluluğunun gönüllü ekibi tarafından hazırlanmaktadır.

6.Projenin hedefi nedir ve ileride ne olacak?
Projenin esas hedefi açık kaynak ve özgür yazılım için farkındalık 
oluşturmaktır. Debian kararlı sürüm dağıtımını takiben yeni sürümlerin 
hazırlanması planlanmaktadır.

7.Projeye nasıl destek olabilirim?
Topluluk sosyal hesaplarımızı takip ederek veya Discord sunucumuza 
katılarak geri bildirimde bulunabilirsiniz. Projeye herhangi bir konuda 
katkıda bulunmak isterseniz iletşim kısmından bize ulaşabilirsiniz.

###########################################################################
Nükleon Linux 1.0 "MUAVENET" Sürümü İçin Yüklü Gelen Paketlerden Bazıları:

Firefox - İnternet Tarayıcı
Gimp - Görüntü Düzenleme Yazılımı
Inkscape - Vektörel Grafik Düzenleme Yazılımı
Blender - 3B Modelleme ve Animasyon Yazılımı
Librecad - 2B Teknik Çizim Hazırlama Yazılımı
Freecad - 3B Teknik Tasarım Yazılımı
Kicad - Elektronik Devre Şeması ve PCB Tasarım Yazılımı
Stellarium - Astronomi Benzetim Yazılımı
Arduino - Arduino IDE Yazılımı
Fritzing - Prototip Şema Tasarım Yazılımı
Audacity - Ses Düzenleme ve Kaydetme Yazılımı
Gnu Octave - MATLAB Benzeri Programlama Yazılımı
Wireshark - Ağ Paketi Çözümleme Yazılımı

###########################################################################
```

</details>

<p></p>

---

## Kaplan 1284 (2019)

<p></p>

<p><img src="/kaplan1284.jpg"/></p>

Kaplan 1284, PCB tasarımını Eagle programı ile yaptığım ve Çin'de üretilen Arduino uyumlu bir geliştirme kartı. Kart, ATMEGA 1284P, 2KB FRAM ve FTDI yongalarını içeriyor. SMD ve DIP bileşenlerin tamamı kalem havya ile lehimlendi.

<details>
<summary><mark>Orjinal blog yazısını göstermek için tıklayın</mark></summary>

<p></p>

## Arduino Uyumlu Geliştirme Kartı Yapımı – Kaplan 1284

Bu yazıda Arduino UNO kartında bulunan mikrodenetleyiciden (ATmega 328p) üstün özelliklere sahip olan ATmega 1284p mikrodenetleyicisinin nasıl Arduino IDE ile programlanabileceğine değineceğiz ve bir geliştirme kartının hazırlanışa göz atacağız.

Arduino UNO kartı, nispeten küçük ve aynı zamanda birçok hobi, kendin yap projelerinin vazgeçilmez cihazı. Ancak biraz daha gelişmiş projelere geçiş yaptığınızda UNO’nun SRAM ve Flash hafızaları size yetersiz gelmeye başlayabilir. Elbette Arduino’nun kolay arayüzünden ve sözdiziminden feragat edip Atmel Studio ile daha özgür programlama yapabilirsiniz, fakat şimdilik bunu bir kenara bırakalım. Arduino UNO kartının sınırlarında gezdiğinizi hissettiğinizde neyse ki imdadınıza Arduino Mega yetişiyor. Mega kartı, ATmega 2560 mikrodenetleyicisinin getirdiği birçok giriş çıkış pini ve yüksek hafıza kapasitesi ile göz dolduruyor. Ama sizin bu kadar çok bağlantı noktasına ve büyük bir karta ihtiyacınız olmayabilir. Form olarak Arduino UNO kartına yakın sayılabilecek boyutlarda, ancak ondan daha güçlü bir kart bazı projeler için oldukça iyi bir çözüm olurdu. İşte tam olarak bu noktada konumlanan bir kartı tasarlayıp üreteceğiz.

|Özellik|Arduino UNO|Kaplan 1284|Arduino Mega|
|-----|-----|-----|-----|
|Flash Hafıza|32K|128K|256K|
|SRAM|2K|16K|8K|
|Dahili EEPROM|1K|4K|4K|
|Harici EEPROM|–|2K FRAM|–|
|UART Donanımı|1|2|4|
|Dijital Pin|14|24|54|
|Analog Pin|6|8|16|
|PWM Pin|6|8|15|

### Önyükleyiciyi Yakmak

Arduino fonksiyonları ve kütüphaneleri ile kullanabilmek için mikrodenetleyicilere, önyükleyici (Bootloader) denilen küçük bir program yüklememiz gerekir. Bu program mikrodenetleyicinin her sıfırlanışında (resetlenmesi) başlangıçta çalışarak asıl programımızı çalıştırır. Önyükleyicisi olmayan mikrodenetleyicimize Arduino ile hazırladığımız kodları doğrudan yükleyemeyiz. Fakat tabi ki mikrodenetleyicimizi, üreticilerin sunmuş olduğu yazılımlar ile programlayıp kullanabiliriz.

ATmega 328p ve ATmega 2560 için önyükleyici programları Arduino IDE’nin içinde halihazırda bulunuyor. Fakat henüz ATmega 1284p mikrodenetleyicili resmi bir Arduino kartı üretilmediği için [bu linkteki](https://github.com/MCUdude/MightyCore) MightyCore önyükleyici ve kütüphane ortamını indirip kullanacağız.

MightyCore kütüphanesini indirip arşivden çıkartıyoruz ve <code>Belgelerim\Arduino\Hardware</code> klasörünün içine kopyalıyoruz. Dosya menüsü altından Örnekler>ArduinoISP örneğini açıp ve bir Arduino UNO kartına yüklüyoruz. Daha sonra aşağıdaki şemada gösterilen devreyi breadboard üzerine kuruyoruz.

<p><img src="/isp.png"/></p>

Araçlar menüsü altından kartı ATmega 1284 olarak seçip, yine aynı menü altındaki programlayıcı seçeneğini ise “Arduino as ISP” olarak değiştiriyoruz. Herşey aşağıdaki gibi görünüyor ise önyükleyiciyi yakmaya hazırız demektir. Bu aşamada önyükleyiciyi yazdır tuşuna basmak yeterli olacaktır.

<p><img src="/bootloader.png"/></p>

Önyükleyici yakma işlemini tamamlandıktan sonra programlayıcı olarak kullandığımız Arduino kartı ile işimiz bitiyor. Bir USB TTL dönüştürücü vasıtasıyla mikrodenetleyicimize doğrudan program yükleyebiliriz. Bu noktada FTDI kartı düşük maliyetli ve pratik yapısıyla işimizi fazlasıyla görecektir. Buraya kadar yaptıklarımızı blink kodunu yükleyerek test edebiliriz. Kodu yüklemeden önce programlayıcıyı tekrar “AVRISP mkII” olarak değiştirmeyi unutmayın. Mighty Core kütüphanesinin standart pin yapısını seçtiğimiz için normalde Arduino UNO’da 13. pine bağlı olan ledimiz burada 0. pine bağlı olacak. MightyCore standart pin yapısının Arduino pin karşılıklarını aşağıdaki şemada görebilirsiniz.

<p><img src="/mightycore.jpg"/></p>

### Ek Özellikler

Yazının girişinde ATmega 1284p mikrodenetleyicisinin yeteneklerine değinmiştik. Şu haliyle bile tasarlayacağımız kart birçok kullanım seneryosu için yeterli olacaktır.

Kaplan 1284 kartımızın mikro USB girişini kartı programlamak ve güç vermek için kullanacağız, ayrı bir güç girişi ve gerilim regületörüne yer vermeyeceğiz. Dahili EEPROM hafızasına ek, yine benzer amaçlarla kullanılabilecek bir harici FRAM entegresi ekleyeceğiz.

FRAM’ler tıpkı EEPROM’lar gibi defalarca yazılıp okunabilen, elektrik enerjisi kesildiğinde hafızasındaki veriyi saklayabilen belleklerdir. FRAM’lerin başlıca avantajları, kısa okuma yazma süreleri, içerisindeki veriyi uzun süre bozulmadan saklayabilmeleri ve çok fazla okuma yazma döngüsüne dayanabilmeleri olarak sayılabilir.

Kartımızda I2C protokolü ile çalışan 2KB bir FRAM kullanıyoruz. Kullandığımız bileşenin detaylı bilgilerine [buradan](https://www.infineon.com/assets/row/public/documents/10/49/infineon-fm24c16b-16-kbit-2-k-8-serial-i2c-f-ram-datasheet-en.pdf) erişebilirsiniz.

<p><img src="/fram.jpg"/></p>

### PCB Tasarımı

PCB tasarlamak için birçok ücretli ve ücretsiz program mevcut. Benim tercihim Eagle’dan yana oldu. KiCad programı da açık kaynaklı bir alternatif olarak kullanılabilir.

Daha önce breadboard üzerine yerleştirip test ettiğimiz bileşenlere ek, header ve ICSP pinleri gibi elemanlar bu aşamada tasarıma dahil oldu. Kartın nihai devre şeması ve üretiminin tamamlanmış haline aşağıdaki görsellerden ulaşabilirsiniz.

<p><img src="/kaplan_sch.png"/></p>

<p><img src="/kaplan1284_render.png"/></p>

<p><img src="/kaplan_uno.jpg"/></p>

</details>

</details>

<p></p>
