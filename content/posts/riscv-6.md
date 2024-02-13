---
title: "RISC-V İşlemci Tasarımı - Bölüm 6: Açık Kaynaklı Yonga Serim Akışı"
date: 2024-02-05
description: Bu bölümde, tasarladığımız işlemci çekirdeğinin yonga serimini gerçekleştiriyoruz.
math: true
---

Önceki bölümlerde FPGA akışını kullanarak işlemcimizi tasarladık ve çalıştırdık. FPGA üzerinde kendi işlemcimizi çalıştırmak her ne kadar heyecan verici olsa da, neticede bütün olaylar başka bir yonganın içinde dönüyor. Bu noktada şu soru akla gelebilir; Peki neden kendi işlemci yongamızı tasarlamayalım?

Açıkçası yakın döneme kadar yonga tasarlamamak için ciddi bahaneler bulabilirdik. Çünkü bu alanda kullanılan ticari elektronik tasarım otomasyonu (Electronic Design Automation - EDA) araçları oldukça yüksek maliyetli ve dışarıya kapalı durumda. Eğer bu yazılımlara erişimi bulunan bir kurumda (üniversite, şirket vb.) değilseniz, yonga tasarımı konularında çalışmak imkansızlaşabiliyordu. Fakat son birkaç yılda "açık kaynaklı donanım/silikon" alanında yaşanan gelişmeler sonucunda öğrenciler, hobiciler ve profesyoneller için tamamen ücretsiz yonga tasarım ve serim araçları ortaya çıkmış oldu.

<p align="center">
  <img src="/asic-and-propiteary-software.png"  width="400"/>
</p>

Bu yazıda, daha önceden tasarlamış olduğumuz Matrak işlemcimizin çekirdeğini açık kaynaklı yonga serim akışından geçirmeyi deneyeceğiz ve elde ettiğimiz çıktıları tartışacağız.

## Yazılım Ortamının Hazırlanması

Yonga serimini gerçekleştirmek için OpenLane'i kullanacağız. OpenLane birçok açık kaynaklı aracı içerisinde barındıran otomatize edilmiş bir akış. Bu akış sayesinde RTL tasarım kodlarımız (verilog, systemverilog vb.) yonga serim çıktısına (GDSII) dönüşüyor.

Şimdi bir de üretim prosesine ihtiyacımız var. PDK (Process Design Kit) kütüphane dosyalarının ilgili yonga fabrikasından bize iletilmesi gerekli. Neyse ki bu noktada imdadımıza açık kaynaklı PDK'lar yetişiyor. SkyWater ve Global Foundries fabrikalarının SKY130 ve GF180 PDK'ları Google sponsorluğunda açık kaynaklı olarak yayınlanmış durumda. Ayrıca başka fabrikalar da açık kaynaklı PDK yayınlıyorlar. Biz bu projede SKY130 prosesine odaklanacağız. SKY130, 5 metal katmanlı 180nm-130nm hibrit üretim süreci olarak [geçiyor](https://github.com/google/skywater-pdk-sky130-raw-data?tab=readme-ov-file#sky130-process-node). OpenLane akışında varsayılan olarak SKY130 PDK kullanılıyor, bu sebeple yalnızca OpenLane'i kurmak bizim için yeterli olacak.

OpenLane kurulumu için Ubuntu veya Windows WSL (Ubuntu 22.04) sistemini kullanabiliriz. Biz WSL üzerinden devam edeceğiz ancak WSL ile doğrudan Ubuntu arasındaki fark yalnızca Docker kurulumu esnasında oluyor.

Evvela Docker'ı bilgisayarımıza kurarak işe koyuluyoruz. Bunun için [buradaki](https://docs.docker.com/desktop/install/windows-install/) resmi rehberi takip edebilirsiniz. Docker'a WSL sistemi içinden erişebilmek için Docker Desktop Settings kısmındaki "Use the WSL 2 based engine" seçeneğinin aktif olduğundan emin olmalıyız.

<p align="center">
  <img src="/docker-wsl.png"/>
</p>

> :warning: Docker'ı bilgisayar başlangıcında çalıştırmayı tercih etmezseniz, OpenLane ile çalışmaya başlamadan önce manuel olarak başlatmak gerektiğini unutmayın.

Docker'ın çalıştığından emin olmak için WSL Bash'i açıp <code>sudo docker run hello-world</code> komutunu verebiliriz. Bu komut, basit bir konteynır çağırıp ekrana "Hello from Docker!" yazdırıyor.

Şimdi OpenLane kurulumuna geçebiliriz. Bash'e şu komutları girerek bazı paketleri yüklemeliyiz.

```
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt install -y build-essential python3 python3-venv make git
```

> :pushpin: Buradaki kurulum rehberi zaman içinde güncelliğini yitirebilir. Resmi kurulum rehberine [buradan](https://openlane.readthedocs.io/en/latest/getting_started/installation/index.html) erişebilirsiniz.

Ardından OpenLane'i klonlayıp kuruyoruz:

```
$ git clone --depth 1 https://github.com/The-OpenROAD-Project/OpenLane.git
$ cd OpenLane/
$ make
```

Şimdi <code>make test</code> komutunu vererek OpenLane'in çalıştığını doğrulayabiliriz. Bu komut SPM örnek projesini akıştan geçirmektedir. Ekranda "Basic test passed" ifadesini gördüğümüzde OpenLane'in doğru çalıştığını anlıyoruz.

## Çekirdeğin Yonga Serimi

<code>OpenLane/desings</code> dizini altında "matrak" isimli bir dizin oluşturuyoruz. Proje dosyalarımız bu dizinde yer alacak. İşlemci çekirdeğimiz olan [matrak.v](https://github.com/necaticakaci/matrak/blob/main/src/matrak.v) dosyasını <code>OpenLane/designs/matrak/src</code> dizinine kopyalıyoruz.

Serim akış ayarları için bir <code>config.json</code> veya <code>config.tcl</code> dosyasına ihtiyacımız var. Burada yalnızca çekirdeğin serimini gerçekleştireceğimiz için tasarımı <code>macro</code> olarak ele alacağız. En sade haliyle aşağıdaki gibi bir <code>config</code> dosyası oluşturabiliriz.

config.json:

```json
{
    "DESIGN_NAME": "matrak",
    "VERILOG_FILES": "dir::src/matrak.v",
    "FP_PIN_ORDER_CFG": "dir::pin_order.cfg",
    "CLOCK_PORT": "clk_i",
    "CLOCK_PERIOD": 20,
    "FP_PDN_MULTILAYER": false,
    "FP_PDN_CORE_RING": false,
    "RT_MAX_LAYER": "met4"
}
```

<details>
<summary>config.tcl: <mark>kodu göstermek için tıklayın</mark></summary>

```tcl
set ::env(DESIGN_NAME) {matrak}
set ::env(VERILOG_FILES) [glob $::env(DESIGN_DIR)/src/matrak.v]
set ::env(FP_PIN_ORDER_CFG) [glob $::env(DESIGN_DIR)/pin_order.cfg]
set ::env(CLOCK_PORT) {clk_i}
set ::env(CLOCK_PERIOD) 20
set ::env(FP_PDN_MULTILAYER) 0
set ::env(FP_PDN_CORE_RING) 0
set ::env(RT_MAX_LAYER) "met4"
```

</details>

<p></p>

<code>DESIGN_NAME</code>: projenin adı

<code>VERILOG_FILES</code>: projenin verilog dosyaları

<code>FP_PIN_ORDER_CFG</code>: giriş çıkış pinlerinin yön ayar dosyası

<code>CLOCK_PORT</code>: tasarımın saat sinyali

<code>CLOCK_PERIOD</code>: tasarımın saat periyodu (20 ns = 50 MHz)

<code>FP_PDN_MULTILAYER</code>: yalnızca alt dikey güç hattı kullanılsın.

<code>FP_PDN_CORE_RING</code>: [güç halkası](https://openlane.readthedocs.io/en/latest/tutorials/digital_guide.html#configure-mem-1r1w) devredışı bırakılsın.

<code>RT_MAX_LAYER</code>: "routing" için kullanılacak en yüksek katmanı sınırlandır.

> :pushpin: <code>FP_PDN_MULTILAYER</code>, <code>FP_PDN_CORE_RING</code> ve <code>RT_MAX_LAYER</code> ayarlarını, tasarımı makro haline getirmek için eklemiş bulunuyoruz.

Giriş çıkış pinlerinin yerleştirileceği yönü belirlemek için "pin_order.cfg" dosyası oluşturuyoruz. Örnek bir tane aşağıdaki gibi olabilir:

pin_order.cfg:

```
#N
inst_i.*
data_i.*

#S
inst_addr_o.*

#E
data_addr_o.*

#W
clk_i
rst_i
stall_i
wen_o
ren_o
stb_o.*
data_o.*
```

Bu üç dosya aşağıda belirtildiği gibi yerleşmeli:

```
designs/matrak
├── config.json
├── pin_order.cfg
├── src
│   ├── matrak.v
```

Herşey doğru gözüküyorsa, OpenLane dizinine gelip <code>make mount</code> komutu ile konteynırı çalıştırıyoruz. Ardından aşağıdaki komutu kullanarak akışı başlatabiliriz.

```
$ ./flow.tcl -design matrak -tag first_run
```

> :pushpin: Akıştan üretilen çıktılar matrak/runs dizini altında yer alıyor. <code>-tag</code> argümanı kullanarak dizine özel isim verebiliyoruz. Bu argüman olmadan akış çalışırsa dizin ismi varsayılan olarak komutun çalıştrıldığı tarih ve saat biçiminde oluyor.

<p align="center">
  <img src="/few-momentos.jpg"/>
</p>

Sabırlı ve endişeli bekleyişin ardından nihayet akış sonuçlanıyor, ve o da ne? :tada: İlk denemede hata almamışız. Birkaç uyarı ile tamamladığımız için oldukça şanslıyız. Üretilen GDS dosyası <code>/matrak/runs/first_run/results/final/gds/matrak.gds</code> dizini altında yer alıyor.

Akışın tamamlanmasının ardından çıktı, rapor ve kayıt dosyalarına <code>runs/first_run</code> dizini altından erişebiliyoruz. <code>reports</code> dizininde bulunan üretilebilirlik raporu "manufacturability.rpt" aşağıdaki gibi oluşturuldu.

manufacturability.rpt:

```
Design Name: matrak
Run Directory: /openlane/designs/matrak/runs/first_run
----------------------------------------

Magic DRC Summary:
Source: /openlane/designs/matrak/runs/first_run/reports/signoff/drc.rpt
Total Magic DRC violations is 0
----------------------------------------

LVS Summary:
Source: /openlane/designs/matrak/runs/first_run/logs/signoff/38-matrak.lef.lvs.log
Number of nets: 8730                       |Number of nets: 8730                       
Design is LVS clean.
----------------------------------------

Antenna Summary:
Source: /openlane/designs/matrak/runs/first_run/logs/signoff/41-arc.log
Pin violations: 40
Net violations: 34
```

Yine <code>reports</code> dizini altında yer alan "metrics.csv" dosyasında akış ile ilgili çeşitli veriler tutuluyor. Bunlardan bazıları aşağıda bulunuyor.

```
Die area (mm^2)         = 0.17
Peak memory usage (MB)  = 854.87
Synth cell count        = 7319
Total cells             = 20887
```

Serim çıktılarını incelemek için OpenRoad GUI'yi kullanabiliriz. Bunun için konteynırın içinde <code>python3 gui.py --viewer openroad ./designs/matrak/runs/first_run/</code> komutunu vermeliyiz.

<p align="center">
  <img src="/matrak-openroad-gui.png"/>
</p>

Klayout aracı ile de GDS dosyamızı görselleştirebiliriz. Bunun için konteynırdan çıkmadan <code>klayout -e -nn $PDK_ROOT/sky130A/libs.tech/klayout/tech/sky130A.lyt -l $PDK_ROOT/sky130A/libs.tech/klayout/tech/sky130A.lyp ./designs/matrak/runs/first_run/results/final/gds/matrak.gds</code> komutunu giriyoruz.

<p align="center">
  <img src="/matrak-gds-klayout.png"/>
</p>

GDS dosyasını görüntülemek için kullanabileceğimiz başka bir yöntem ise "gdstk" Python kütüphanesini kullanmak. Bunun için <code>python3 gds2svg.py matrak.gds</code> komutu ile aşağıdaki kodu çalıştırabiliriz.

gds2svg.py:

```python
import gdstk
import sys
import os

if len(sys.argv) < 2:
    print("Usage: python gds2svg.py <file_path>")
    sys.exit(1)

file_path = sys.argv[1]
file_name = os.path.basename(file_path)
file_name_split = os.path.splitext(file_name)[0]
file_name_out = f"{file_name_split}.svg"

gds = gdstk.read_gds(file_path)
cells = gds.top_level()
cells[0].write_svg(file_name_out)
```

> :bulb: Akıştan üretilen GDS dosyaları genellikle büyük boyutlarda, dolayısıyla üretilen SVG dosyası da (açmaya çalıştığınızda bilgisayarınızda kilitlenmelere neden olabilecek kadar) büyük oluyor. Bu nedenle SVG dosyalarını Cairosvg ile PNG dosyalarına dönüştürmek iyi fikir olabilir.

<p align="center">
  <img src="/matrak-gdstk.png"/>
</p>

Matrak çekirdeğini OpenLane akışından geçirdik. Fakat tasarımda henüz bellek ve çevrebirimleri yer almadığı için "işlemci" yongamızı tamamlamış sayılmıyoruz.
