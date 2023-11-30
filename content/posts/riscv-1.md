---
title: "RISC-V İşlemci Tasarımı - Bölüm 1: Matrak"
date: 2023-05-22
description: Bu bölümde RISC-V buyruk kümesi mimarisine hızlı bir giriş yaparak, birkaç buyruk destekleyen işlemci tasarlıyoruz.
math: true
---

İşlemcilerin nasıl çalıştığını anlamanın en iyi yollarından biri işlemciyi tasarlamak. 32-bit RISC-V (RV32I) mimarisi öğrenmek için en uygun ve kolay işlemci mimarileri arasında yer alıyor. Bu yazı dizisinde Verilog donanım tanımlama dili ile sıfırdan bir RISC-V işlemci tasarlayacağız. Tasarladığımız işlemciyi hem benzetim ortamında hem de FPGA üzerinde deneyeceğiz. İlk defa işlemci tasarlayacaklar veya hobi olarak ilgilenenler için kolaylık olması arzusuyla işleri oldukça basit tutmaya çalışacağız. Fakat basit olması fonksiyonel olmayacağı anlamına gelmiyor, nitekim C dili ile örnek programlar hazırlayıp tasarladığımız işlemcinin üzerinde çalıştıracağız. Görüş ve önerilerinizi yorumlar kısmından veya [iletişim](/about) kısmından aktarabilirsiniz. Faydalı olması dileğiyle\.\.\.

Yazıda geçen raptiye (:pushpin:) ile belirtilen kısımlar ilave not anlamına gelirken lamba (:bulb:) ile gösterilen kısımlar iyi fikir manasına gelmektedir.

Başlamadan önce sayısal tasarım, bilgisayar mimarisi ve Verilog konularına biraz hakim olmanın oldukça faydalı olacağını belirtelim. Ve tabii ki isim önemli, tasarlayacağımız işlemciyi "Matrak" olarak adlandırdık, bundan sonra bu isimle çağıracağız. Matrak işlemcimiz ilk etapta RV32I mimarisine sahip olacak ve bu en yalın haliyle bir RISC-V işlemcisi demek oluyor.

> :pushpin: Esasında RV32E olarak adlandırılan, RV32I buyruk kümesinin bir alt kümesi daha var. Fakat bu mimariyi daha az yaygın olması ve derleyici desteğinin kısıtlı olması nedeniyle tercih etmedik.

> :bulb: Matrak işlemcimizin tüm kodlarına [GitHub reposundan](https://github.com/necaticakaci/matrak) erişebilirsiniz.

Biz kısaca işlemci olarak bahsetsek de aslında tasarlamamız gereken şey sadece bir işlemci değil, birlikte çalışabilecek komple bir sistem. Bu sistemlerde genelde işlemciye ek olarak bellekler ve muhtelif çevrebirimleri yer alır. Bu sistemler tek yonga üzerinde toplanırsa buna System on Chip (SoC) adı verilir. Bizim tasarlayacağımız sistem ise şimdilik bir işlemci ve bir bellekten meydana gelecek.

<p align="center">
  <img src="/cpu-memory.png"/>
</p>

<p align="center">
  Sistem üst modül şeması
</p>

## Bellek Tasarımı

İşlemci tasarımına geçmeden önce ana bellek olarak kullanmak için bir RAM bellek modülü yazalım. 2 KiB boyutunda bir bellek şimdilik işimizi fazlasıyla görecektir. İlk olarak bellek modülünün giriş ve çıkışlarını tanımlıyoruz. <code>clk_i</code> saat sinyali, <code>wen_i</code> ise yazma yetkilendirme girişidir. Bellekten okunan veri <code>data_o</code> çıkışından aktarılır, belleğe yazılacak veri <code>data_i</code> girişinden alınır. <code>addr_i</code> girişinden okunacak veya yazılacak verinin adresi gelir.

```verilog
module memory (
   input          clk_i,
   input          wen_i,
   input [31:0]   addr_i,
   input [31:0]   data_i,
   input [31:0]   data_o
);
```

32-bit genişlikte 512 satır hafıza alanı tanımlıyoruz. Adres bağlantısını 32-bit olarak tanımladık fakat 512 satır için esasında 10-bit yeterlidir.

```verilog
   // 512x32 bit = 16384 bit = 2048 bayt = 2 kibibayt (2 KiB)
   reg [31:0] mem [511:0];
```

Aşağıda belleğin okuma ve yazma işlemlerini tanımlıyoruz. Okuma asenkron (saat sinyalinden bağımsız) olarak yapılırken, yazma işlemi saatin yükselen kenarında <code>wen_i</code> sinyalinin 1 olması durumunda gerçekleştirilmektedir. Belleğimizde tek satırda 32-bit yani 4 bayt bulunduğundan dolayı adresin en anlamsız 2 bitini kullanmıyoruz.

```verilog
   // Asenkron okuma
   assign data_o = mem[addr_i[31:2]];

   // Senkron yazma
   always @(posedge clk_i) begin
      if (wen_i) begin
         mem[addr_i[31:2]] <= data_i;
      end
   end

endmodule
```

İlerleyen bölümlerde belleğe geri dönüp düzenlemeler yapacağız fakat şimdilik bellek tasarımı tamamlandı ve işlemci tasarımına geçmeye hazırız.

## İşlemci Mimarisi

İlk olarak söylenmesi gereken şey işlemciler aynı buyruk kümesini destekleseler dahi, bu işlemcilerin mikromimari açısından birbirinin aynısı olduğu anlamına **gelmez**. Dolayısıyla bir RISC-V işlemciyi farklı hedefleri önceleyerek değişik şekillerde tasarlayabiliriz. Bizim durumumuzda önceliğimiz anlaşılabilirlik ve basitlik olacak. En temel haliyle bir işlemci tek vuruşluk olarak tasarlanabilir. Tek vuruşluk ifadesi, işlemcinin bir saat vuruşunda (çevriminde) bir buyruğu işlemesi manasına gelmektedir. Bu kulağa güzel geliyor ama aynı zamanda bu bellekten okunan bir buyruğun işlemcinin (devreleri) içinde baştan sona "gezmesi" demek oluyor. Yani yol ne kadar uzarsa o kadar gecikme oluşacak. Sinyallerin gecikmeleri ise saat frekansımızın düşmesine sebep olarak işlemcimizin başarımını düşürüyor. Bu duruma çare olarak çok vuruşluk işlemci mimarisini sayabiliriz. Çok vuruşluk işlemcilerde, buyruğun geçeceği aşamalar bölünür ve bu sayede daha yüksek saat frekanslarına çıkılabilir. Ama buyruklar artık birden fazla saat çevriminde tamamlanır. Boru hattı mimarileri ise hem işlemciyi aşamalara bölmeyi hem de bu bölümlerin verimli kullanılmasını sağlar. Ancak boru hattı mimarileri beraberinde bazı çözülmesi gereken sorunları getirir. Boru hattı mimarisi, işlemci aşamalarının sürekli yeni buyrukla beslenmesini sağlayarak çevrim başına bir buyruk işlemeye yaklaşır, buna karşılık boru hattı sorunlarının çözümü mimari tasarımında zorluklara yol açar.

Çok vuruşluk işlemciler nispeten kolay tasarımları nedeniyle hobi ve eğitim amaçlı projelerde sıklıkla kullanılır. Fakat biz Matrak işlemcimizde 2 aşama boru hattı tasarımı tercih edeceğiz. Hem ideal durumda çevrim başına 1 buyruk (en kötü durumda: 2 çevrimde 1 buyruk) işleyeceğiz, hem de daha fazla aşamalı boru hattı mimarilerin getirdiği sorunlarla yüzleşmeyeceğiz.

<p align="center">
  <img src="/cpu-architecture.png"/>
</p>

<p align="center">
  Tasarlayacağımız işlemcinin basitleştirilmiş şeması
</p>

### Getirme (Fetch) Birimi

İlk olarak getirme birimini tasarlıyoruz. Getirme biriminin görevi, sıradaki buyruğu bellekten okumaktır. Bellekten okunacak buyruğun adresi program sayacı (program counter - PC) adı verilen bir kaydedicide tutulur. Bu kaydedicinin çıkışı program belleğine adres olarak bağlanır.

>:pushpin: Program sayacına, buyruk işaretçisi (instruction pointer) ve buyruk adres kaydedicisi (instruction address register) de denilmektedir.

Matrak işlemcisinde program sayacı, asenkron sıfırlamalı D tipi 32 adet flip-flop'tan meydana geliyor. Sıfırlama sinyali geldiğinde program sayacına başlangıç adres değeri yüklenir. Bu adreste programın ilk buyruğu bulunur. Biz başlangıç adresini 0 olarak belirledik ancak farklı bir adresten de başlatmak mümkündür.

Belleği bir adres satırında 32-bit yer alacak şekilde tasarlamıştık. RISC-V mimarisinde buyruklar 32-bit genişliktedir. (İstisnai durum: C eklentisi 16-bit). Bu sebeple sonraki buyruğa erişmek için program sayacını 4 artırarak gitmeliyiz. Tasarladığımız mimaride her saat çevriminde yeni bir buyruğu işleme alacağız, dolayısıyla program sayacı saat sinyalinin her yükselen kenarında 4 arttırılmaktadır.

```verilog
module fetch (
   input                clk_i,
   input                rst_i,
   output reg [31:0]    pc_o
);

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin // Sıfırlama sinyali geldiyse program sayacına başlangıç adresini ata
         pc_o <= 32'h0000_0000;
      end else begin // Saat sinyali geldiyse program sayacına 4 ekle
         pc_o <= pc_o + 4;
      end
   end

endmodule
```

<p align="center">
  <img src="/fetch.png"/>
</p>

<p align="center">
  Getirme modülü şeması
</p>

### Boru Hattı Kaydedicisi (Pipeline Register)

Daha önce de değindiğimiz gibi çalışma frekansını biraz daha yükseltmek için boru hattı kaydedicisi ekleyerek işlemciyi iki parçaya böleceğiz. Teorik olarak daha çok boru hattı aşaması daha yüksek çalışma frekansı sağlasa da boru hattı aşamasını artırmanın tasarımda karmaşıklığı arttırıcı etkisi olacaktır. Matrak mimarisindeki iki aşamalı boru hattı ile dallanma ve veri bağımlığı gibi sorunlarla uğraşmayacağız. Boru hattının birinci aşamasında getirme işlemi, ikinci aşamasında ise çözme ve yürütme işlemleri yapılacak.

```verilog
module fd_regs (
   input                clk_i,
   input                rst_i,
   input [31:0]         inst_f_i, // Buyruk sinyali (bellekten geliyor)
   output reg [31:0]    inst_d_o  // Boru hattı kaydedicisinin çıkışı 
);

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         inst_d_o <= 32'b0;
      end else begin
         inst_d_o <= inst_f_i;
      end
   end

endmodule
```

Boru hattı kaydedicisi şimdilik bellekten okunan buyruğu tutacak. Boru hattı kaydedicisi 32 adet D tipi flip-floplardan  meydana geliyor.

<p align="center">
  <img src="/fd_regs.png"/>
</p>

<p align="center">
  Boru hattı kaydedicisi
</p>

### Çözme (Decode) Birimi

Boru hattı kaydedicisinin çıkışından gelen buyruk, çözme birimine ulaşır ve burada hangi genel amaçlı kaydedicileri kullanacağı çözülür. Çözme birimimizin içinde 32-bit genişlikte 31 adedi genel amaçlı olmak üzere toplam 32 kaydedici yer alıyor. İstisnai olan x0 kaydedicisi sabit 0 değerini barındırıyor ve değeri değiştirilemiyor.

Kaydedicilerin her ne kadar genel amaçlı olduğunu söylesek de aslında bir kullanım standardı var ve bu standart ABI (Application Binary Interface) olarak adlandırılıyor. Örneğin C/C++ gibi bir programlama dili ile yazdığımız program derlenirken buyruklarda kullanılacak kaydediciler ABI'de tanımlı görevine uygun biçimde seçiliyor. Tabii bunun yalnızca bir standart arayüz oluşturmak için yapıldığını tekrarlayalım, neticede x0 haricindeki kaydedicilerin birbirlerinden hiçbir farkı bulunmuyor. Aşağıdaki tabloda kaydedicilerin ABI karşılıkları ve görev tanımları gösterilmiştir.

| Kaydedici | ABI Adı | Tanım |
| --------- | ------- | ----- |
|x0         |zero     |hardwired zero|
|x1         |ra       |return address|
|x2         |sp       |stack pointer|
|x3         |gp       |global pointer|
|x4         |tp       |thread pointer|
|x5         |t0       |temporary register 0|
|x6         |t1       |temporary register 1|
|x7         |t2       |temporary register 2|
|x8         |s0/fp    |saved register 0 / frame pointer|
|x9         |s1       |saved register 1|
|x10        |a0       |function argument 0 / return value 0|
|x11        |a1       |function argument 1 / return value 1|
|x12        |a2       |function argument 2|
|x13        |a3       |function argument 3|
|x14        |a4       |function argument 4|
|x15        |a5       |function argument 5|
|x16        |a6       |function argument 6|
|x17        |a7       |function argument 7|
|x18        |s2       |saved register 2|
|x19        |s3       |saved register 3|
|x20        |s4       |saved register 4|
|x21        |s5       |saved register 5|
|x22        |s6       |saved register 6|
|x23        |s7       |saved register 7|
|x24        |s8       |saved register 8|
|x25        |s9       |saved register 9|
|x26        |s10      |saved register 10|
|x27        |s11      |saved register 11|
|x28        |t3       |temporary register 3|
|x29        |t4       |temporary register 4|
|x30        |t5       |temporary register 5|
|x31        |t6       |temporary register 6|

>:pushpin: Bir kaydedicinin değerinin hep 0 olması kulağa garip gelebilir, esasında bu durum RISC-V mimarisinde yapılan önemli tercihlerden biri. Bu sayede mimaride donanımsal olarak yer almayan bazı buyruklar sözde buyruk (psuedo instruction) olarak kullanılabiliyor. Örneğin <code>nop</code> sözde buyruğu, mimaride tanımlı <code>addi x0, x0, 0</code> eşlenik buyruğuna dönüştürülüyor. Eşlenik buyruktan da anlaşılacağı üzere <code>nop</code> buyruğu herhangi bir eylem gerçekleştirmiyor. Ayrıca RISC-V tasarımcıları sözde buyrukları kullanarak daha az sayıda "gerçek buyruk" kullanmayı sağlamışlar. Bu durum tasarlanan işlemcilerin karmaşıklığının azaltılmasına olumlu yansıyor.

>:bulb: Sözde buyruklar mimariyi sabit tutarak geliştiricilere daha fazla buyruk sağlamak için kullanılır. Bu sayede assembly programcısı diğer işlemci mimarilerinden aşina olduğu bazı buyrukları kullanabilir. Sözde buyrukların bir diğer faydası ise birden fazla buyruğu arka arkaya ekleyerek kodu yalınlaştırmaktır. Programcı için daha anlaşılır olan sözde buyruklar, derlenirken işlemcinin halihazırda tanıdığı eşlenik buyruklara dönüştürülür.

Peki bir RISC-V buyruğu neye benziyor? Örnek olarak toplama işlemimini gerçekleştiren "add" buyruğunu ele alalım.

<p align="center"><code>add rd, rs1, rs2</code> <b>-></b> <code>rd = rs1 + rs2</code></p>

add buyruğu rs1 kaydedicisinin değeri ile rs2 kaydedicisinin değerini toplayıp rd kaydedicisine yazar. Buyrukta yer alan rd, rs1 ve rs2 kısımlarına herhangi bir RISC-V kaydedicisi getirilebilir.

RV32I mimarisi için tanımlanmış toplamda 6 buyruk formatı bulunuyor. Biz şimdilik sadece R-type ve I-type formatlarıyla ilgileneceğiz. Aşağıdaki tabloda buyruk formatları ve ilgili bit aralıkları verilmiştir.

| Bitler  | 31 - 25 | 24 - 20 | 19 - 15 | 14 - 12 | 11 - 7  | 6 - 0   |
| ------- | ------- | ------- | ------- | ------- | ------- | ------- |
| R-type  | funct7  | rs2     | rs1     | funct3  | rd      | opcode  |
| I-type  | imm[11:0]        || rs1     | funct3  | rd      | opcode  |

İki buyruk formatına da baktığımızda bit aralıklarının çoğunlukla aynı amaçla kullanıldığı gözümüze çarpıyor. İlk 7 bit her iki formatta da -ve henüz değinmediğimiz formatlarda da- "opcode" yani işlem kodu olarak kullanılıyor. iki formatta da 11. ve 7. bitlerin arasındaki 5 bit "rd" yani hedef kaydedicisinin adresini tutuyor. 12. ve 14. bitlerin arasında 3 bitlik "funct3" kısmı yer alıyor. Bu alan aynı "opcode"a sahip farklı buyruklar olmasını sağlıyor. 15. ve 19. bitlerin arasında 5 bit "rs1" yani birinci kaynak kaydedicisinin adresini tutuyor. 20. ve 31. bitlerin arasında R-type buyruk formatında "rs2" yani ikinci kaynak kaydedicisi ve "funct7" alanlarını görürken I-type formatında bu alanda 12 bitlik "imm" yani ivedi değer alanını görüyoruz.

>:bulb: Farklı buyruk formatlarındaki ortak amaçla kullanılan alanlar çözme birimini basitleştirmektedir.

R-type formatta iki kaynak kaydedicisindeki değer, buyrukta tanımlı işlemden geçirilip hedef kaydedicisine yazılır. Buyruktaki işlemin ne olduğu opcode, funct3, funct7 kısımlarına bakılarak anlaşılır. I-type formatta ise bir kaynak kaydedicisi ve buyruğa gömülen 12-bit sayı (ivedi değer), buyrukta tanımlı işlemden geçirilip hedef kaydedicisine yazılır. Buyrukta tanımlanan işlem opcode ve funct3 kısımlarından anlaşılır.

Peki RV32I mimarisinde R-type ve I-type formatlarında hangi buyruklar var?

**R-type Buyruklar:**

| Buyruk          | Tanım                             | opcode          | funct3          | funct7          |
| --------------- | --------------------------------- | --------------- | --------------- | --------------- |
| add             | rd = rs1 + rs2                    | 0110011         | 0x0             | 0x00            |
| sub             | rd = rs1 - rs2                    | 0110011         | 0x0             | 0x20            |
| xor             | rd = rs1 ^ rs2                    | 0110011         | 0x4             | 0x00            |
| or              | rd = rs1 \| rs2                   | 0110011         | 0x6             | 0x00            |
| and             | rd = rs1 & rs2                    | 0110011         | 0x7             | 0x00            |
| sll             | rd = rs1 \<\< rs2                 | 0110011         | 0x1             | 0x00            |
| srl             | rd = rs1 \>\> rs2                 | 0110011         | 0x5             | 0x00            |
| sra             | rd = rs1 \>\>\> rs2               | 0110011         | 0x5             | 0x20            |
| slt             | rd = (rs1 < rs2) ? 1:0            | 0110011         | 0x2             | 0x00            |
| sltu            | rd = unsigned(rs1 < rs2) ? 1:0    | 0110011         | 0x3             | 0x00            |

**I-type Buyruklar:**

| Buyruk          | Tanım                             | opcode          | funct3          | funct7          |
| --------------- | --------------------------------- | --------------- | --------------- | --------------- |
| addi            | rd = rs1 + imm                    | 0010011         | 0x0             |                 |
| xori            | rd = rs1 ^ imm                    | 0010011         | 0x4             |                 |
| ori             | rd = rs1 \| imm                   | 0010011         | 0x6             |                 |
| andi            | rd = rs1 & imm                    | 0010011         | 0x7             |                 |
| slli            | rd = rs1 \<\< imm[4:0]            | 0010011         | 0x1             | 0x00            |
| srli            | rd = rs1 \>\> imm[4:0]            | 0010011         | 0x5             | 0x00            |
| srai            | rd = rs1 \>\>\> imm[4:0]          | 0010011         | 0x5             | 0x20            |
| slti            | rd = (rs1 < imm) ? 1:0            | 0010011         | 0x2             |                 |
| sltiu           | rd = unsigned(rs1 < imm) ? 1:0    | 0010011         | 0x3             |                 |

> :pushpin: Çıkarma işlemi buyruğu olan sub yalnızca R-type formatında mevcut, çünkü I-type buyruk formatında çıkarma işlemi addi buyruğu ile yapılabiliyor. Addi buyruğu ile çıkarma yapmak için ivedi değerin negatif bir sayı olması yeterli.

> :bulb: RISC-V buyruk kümesinde bir kaydediciye ivedi değer yüklemek için ayrı bir buyruk bulunmuyor. Bu işlem li sözde  buyruğu ile veya addi buyruğu ile gerçekleştirilebilir. Örn: <code>li a0, 0x45</code> veya <code>addi a0, x0, 0x45</code> 

> :pushpin: Bir kaydediciye 32 bit sayıyı ivedi olarak yazmak için addi buyruğu ile birlikte lui buyruğu kullanılıyor. addi buyruğu ile ivedi değerin 12 bitlik kısmı, lui buyruğu ile kalan 20 bitlik kısmı aktarılıyor. lui buyruğu şimdilik konumuzun dışında olduğu için ona daha sonra değineceğiz.

> :pushpin: Diğer I-type buyrukların aksine slli, srli ve srai buyruklarının funct7 alanları var. Bu buyruklar ivedi değerin yalnızca 5 bitini kullanıyor.

I-type buyrukların ivedi değerlerinin 12 bit genişlikte olduğunu biliyoruz ancak işlemcimiz 32 bit sayılarla işlem yapacak. Bu yüzden buyruktan gelen ivedi değerleri 32 bite genişletmeliyiz. İvedi sayıyı 32 bite genişletmek için 31. ve 12. bitlerin arasındaki 20 bitlik boşluğu, sayının en anlamlı (MSB) biti ile dolduracağız. Bu sayede ivedi değerin işareti kaybolmamış olacak. Bu işlemi çözme biriminin içinde halledeceğiz.

Birçok önemli konuya değindiğimize göre nihayet çözme birimimizin tasarımına geçebiliriz. Çözme birimimizde olması gerekenlere tekrar bir göz atalım:

* Buyrukta yer alan kaydedici adreslerinin çözümlenmesi
* 32 adet 32 bit kaydedicili kaydedici dosyası (register file)
* İvedi genişletici

Yukarıda belirtilen isterlere göre aşağıdaki gibi bir verilog modülü şimdilik işimizi görecektir.

```verilog
module decode (
   input                   clk_i,
   input                   regfile_wen_i, // Kaydedici dosyası yazma yetkilendirme
   input [2:0]             imm_ext_sel_i, // İvedi genişletici format seçimi
   input [31:0]            inst_i,        // Boru hattı kaydedicisinden gelen buyruk
   input [31:0]            result_i,      // Hedef kaydedicisine (rd) yazılacak değer 
   output [31:0]           reg_a_o,       // Birinci kaynak kaydedicisinin (rs1) değeri
   output [31:0]           reg_b_o,       // İkinci kaynak kaydedicisinin (rs2) değeri
   output reg [31:0]       imm_ext_o      // İvedi genişleticinin çıkışı
);

   // 32 bit genişlikte 32 adet kaydedicili kaydedici dosyası
   reg [31:0] regfile [31:0];

   // Kaydedici adreslerini buyruktan ayıkla
   wire [4:0] reg_a_addr      = inst_i[19:15];  // rs1 adres
   wire [4:0] reg_b_addr      = inst_i[24:20];  // rs2 adres
   wire [4:0] target_reg_addr = inst_i[11:7];   // rd adres

   // Kaydedici dosyasından oku
   assign reg_a_o = (reg_a_addr == 5'b0) ? 32'b0 : regfile[reg_a_addr]; // rs1 değeri
   assign reg_b_o = (reg_b_addr == 5'b0) ? 32'b0 : regfile[reg_b_addr]; // rs2 değeri

   // Kaydedici dosyasına yaz
   always @(posedge clk_i) begin
      if (regfile_wen_i) begin
         regfile[target_reg_addr] <= result_i;
      end
   end

   // İvedi genişletici
   always @(*) begin
      case (imm_ext_sel_i)
         3'b000   : imm_ext_o = {{20{inst_i[31]}}, inst_i[31:20]};
         default  : imm_ext_o = 32'b0; 
      endcase
   end

endmodule
```

Hatırlarsanız boru hattı kaydedicisi bellekten okunan buyruğu tutuyordu. Boru hattı kaydedicisinin çıkışını çözme birimimizin <code>inst_i</code> girişine bağlayacağız, dolayısıyla buradan buyruk gelecek. Buyrukta yer alan kaynak kaydedici (rs1 ve rs2) adreslerini alıp kaydedici dosyasından okuyoruz. Bu işlem asenkron olarak gerçekleşiyor. Hedef kaydedici (rd) adresine de geriyazma biriminden gelecek olan 32 bit işlem sonucunu yazıyoruz. Yazma işleminin gerçekleştirilmesi için saatin yükselen kenarında <code>regfile_wen_i</code> sinyalinin 1 olması lazım. Bu sinyal daha sonra yazacağımız kontrol birimi modülünden gelecek. İvedi genişletici, 12 bit ivedi değeri 32 bit haline getirip <code>imm_ext_o</code> çıkışına yazıyor. Yine benzer şekilde ivedi formatı seçecek <code>imm_ext_sel_i</code> sinyali kontrol biriminden gelecek.

> :pushpin: Sıfırıncı adreste bulunan x0 kaydedicisinin her zaman sıfıra bağlı olması özel durumundan bahsetmiştik. Biz burada kaydedici dosyasını ram bellek gibi tasarladık. Özel durumu sağlamak için ise kısmen hile yapıyoruz. Kaydedici adres girişi 0 ise çıkışa 0 aktarılıyor, adres girişi 0 değilse kaydedicinin değeri aktarılıyor. Özetle x0 kaydedicimize aslında değer yazabiliyoruz ancak okunan veri her daim 0 oluyor.

> :pushpin: Henüz R-type ve I-type haricindeki formatlara değinmedik ancak diğer formatlarda ivedi değerler buyruğun farklı alanlarında olabiliyor. Farklı formatları desteklemek için <code>imm_ext_sel_i</code> sinyali ile seçilen bir mux (çoklayıcı) yapısı oluşturduk. Aslında bir case yapısı yerine <code>assign imm_ext_o = {{20{inst_i[31]}}, inst_i[31:20]};</code> yazmak şimdilik yeterliydi.

> :pushpin: Çözme birimimizde buyruğun ne olduğuyla ilgilenmiyoruz. Bu görev daha sonra yazacağımız kontrol birimine ait olacak.

### ALU

Şimdiye kadar neler yaptığımızı bir toparlayalım. İlk olarak getirme birimini tasarladık, getirme birimi sıradaki buyruk adresini hesaplayıp belleğe iletiyordu. Bellekten okunan buyruk ise boru hattı kaydedicimizde tutuluyordu. Son tasarladığımız çözme birimi ise buyruktan hangi kaydedicilerin kullanılacağını anlayıp ilgili kaydedicilerin değerlerini ve genişletilmiş ivedi değeri çıkış olarak aktarıyordu.

ALU yani aritmetik mantık birimi (arithmetic logic unit) işlemcinin aritmetik ve mantıksal işlemleri gerçekleştirdiği birimidir. Biz ALU'muzda şimdilik sadece toplama, çıkarama, VE, VEYA, XOR buyrukların R-type ve I-type karşılıklarına yer vereceğiz. R-type buyruklarla işlem yapılırken rs1 ve rs2 kaydedicilerin değerleri kullanılır. Bu yüzden R-type buyruklarda ivedi genişleticinin çıkışını önemsemeyeceğiz. I-type buyruklarda da rs1 ve ivedi değer ile işlem yapılacağı için rs2 değerini önemsemeyeceğiz. Seçimi tahmin edebileceğiniz gibi bir mux ile gerçekleştireceğiz ve seçim sinyali yine kontrol biriminden gelecek. Kontrol birimi ilgili buyruğa bakarak ALU'da yapılması gereken işlemi seçecek.

Tasarladığımız ALU'nun verilog kodu aşağıdadır.

```verilog
module alu (
   input                      alu_sel_i,  // İkinci işlenenin seçim sinyali (rs2 veya imm)
   input [3:0]                alu_fun_i,  // İşlem seçim sinyali
   input [31:0]               reg_a_i,    // rs1 değeri
   input [31:0]               reg_b_i,    // rs2 değeri
   input [31:0]               imm_ext_i,  // imm değeri
   output reg [31:0]          alu_out_o   // sonuç değeri
);

   // Birinci işlenen iki buyruk formatında da sabit.
   wire signed [31:0] alu_a = reg_a_i;
   // İkinci işlenen seçim sinyaline göre belirleniyor.
   wire signed [31:0] alu_b = alu_sel_i ? imm_ext_i : reg_b_i;

   always @(*) begin
      case (alu_fun_i)
         4'b0000  : alu_out_o = alu_a + alu_b;  // Toplama 
         4'b0001  : alu_out_o = alu_a - alu_b;  // Çıkarma
         4'b0010  : alu_out_o = alu_a & alu_b;  // VE
         4'b0011  : alu_out_o = alu_a ^ alu_b;  // XOR
         4'b0100  : alu_out_o = alu_a | alu_b;  // VEYA
         default  : alu_out_o = 32'bx;          // Geçersiz alu_fun_i sinyali
      endcase
   end

endmodule
```

### Geriyazma (Writeback) Birimi

Geriyazma birimimiz şimdilik sadece ALU'dan aldığı sonucu çözme birimine aktarmakla görevli. Çözme birimi de sonucu rd kaydedicisine yazacak ve R-type, I-type formatındaki buyrukların işlenmesi tamamlanmış olacak.

```verilog
module writeback (
   input  [31:0]              alu_out_i,
   output [31:0]              result_o
);

   assign result_o = alu_out_i;
   
endmodule
```

> :pushpin: Diğer birimleri düşündüğümüzde, geriyazma biriminin biraz boynu bükük kalmış gibi görülebilir. İlerleyen bölümler için bir "spoiler" vermek gerekirse; geriyazma biriminine başka girişler bağlanacağını ve bir mux yardımıyla bunların arasından seçim yapılıp, çıkışa aktarılacağını söyleyebiliriz.

### Kontrol Birimi

ALU'yu işlemcinin kalbi olarak düşünürsek kontrol birimi de işlemcinin beynidir. Öyle ki bu birim, buyruğun ne olduğunu anlamalı ve gerekli kontrol sinyallerini diğer birimlere iletmelidir. Matrak işlemcimizin kontrol birimi kodu aşağıda verilmiştir.

```verilog
module controller (
   input [31:0]               inst_i,        // Boru hattı kaydedicisinden gelen buyruk
   output                     regfile_wen_o, // Kaydedici dosyası yazma yetkilendirme sinyali
   output [2:0]               imm_ext_sel_o, // İvedi genişletici format seçim sinyali
   output                     alu_sel_o,     // ALU ikinci işlenen seçim sinyali
   output reg [3:0]           alu_fun_o      // ALU işlem seçim sinyali
);

   // Buyruğun gerekli bölümleri ayıklanıyor.
   wire [6:0] opcode = inst_i[6:0];
   wire [2:0] funct3 = inst_i[14:12];
   wire [6:0] funct7 = inst_i[31:25];

   wire [1:0] alu_dec;

   reg [6:0] control_signals;
   assign {regfile_wen_o, imm_ext_sel_o, alu_sel_o, alu_dec} = control_signals;

   always @(*) begin
      case (opcode)  // Opcode'a göre kontrol sinyallerinin değerleri belirleniyor. 
         7'b0110011  : control_signals = 7'b1_xxx_0_11; // R-type buyruk
         7'b0010011  : control_signals = 7'b1_000_1_11; // I-type buyruk
         7'b0000000  : control_signals = 7'b0_000_0_00; // Sıfırlama durumu
         default     : control_signals = 7'bx_xxx_x_xx; // Geçersiz buyruk
      endcase
   end

   // Buyruk R-type ise ve funct7 değeri 0x20 ise çıkarma işlemi anlamına gelir.
   wire sub = opcode[5] & funct7[5];

   // ALU'da yapılacak işlem belirleniyor.
   always @(*) begin
      case (alu_dec)
         2'b11    : // R-type veya I-type
            case (funct3)
               3'b000   : // add-addi veya sub buyruğu
                  if (sub) begin
                     alu_fun_o = 4'b0001; // sub
                  end else begin
                     alu_fun_o = 4'b0000; // add, addi
                  end
               3'b100   : alu_fun_o = 4'b0011; // xor, xori
               3'b110   : alu_fun_o = 4'b0100; // or, ori
               3'b111   : alu_fun_o = 4'b0010; // and, andi
               default  : alu_fun_o = 4'b0000;
            endcase
         default  : alu_fun_o = 4'b0000; // Varsayılan işlemi toplama olarak belirledik.
      endcase
   end

endmodule
```

Farzedelim ki bellekten <code>addi x6, x5, 0x16</code> buyruğu okundu. Boru hattı kaydedicisinde tutulan buyruk bir sonraki çevrimde ikinci aşamaya ulaşır. Buyruk bu noktada, çözme birimine ve kontrol birimine gelmiş olur. Kontrol biriminde buyruğun opcode'una bakılır ve bunun bir I-type formatlı buyruk olduğu anlaşılır. Bu durumda kontrol sinyalleri şöyle değişir:

* <code>regfile_wen_o</code>: 1
* <code>imm_ext_sel_o</code>: 000
* <code>alu_sel_o</code>: 1
* <code>alu_dec</code>: 11

> :pushpin: <code>regfile_wen_o</code> sinyali 1 olmalı çünkü x6 hedef kaydedicisine toplama sonucu yazılacak.

> :pushpin: I-type ivedi formatının seçilmesi için 3 bitlik <code>imm_ext_sel_o</code> sinyali 0 olmalı.

> :pushpin: <code>alu_sel_o</code> sinyali 1 olmalı çünkü x5 kaydedicisi ile **bir ivedi sayı** olan 0x16 toplanacak.

> :pushpin: R-type ve I-type formatlarda ALU'da yapılacak işlem, buyruktaki funct3 ve funct7 bitlerine göre değerlendirileceği için <code>alu_dec</code> sinyali 11 olmalı.

Çözme biriminden x5 kaydedicisinin değeri ve 0x16 ivedi değeri ALU'ya aktarılır. ALU'da birinci işlenen olarak x5 kaydedicisinin değeri ikinci işlenen olarak 0x16 değeri toplanır ve sonuç geri yazma birimine aktarılır. Çözme birimi geriyazma biriminden sonucu alır ve aynı saat çevriminde x6 kaydedicisine yazar. Böylelikle <code>addi x6, x5, 0x16</code> buyruğunun serüveni tamamlanmış olur.

> :pushpin: Boru hattı kaydedicisine sıfırlama sinyali geldiğinde kontrol biriminin <code>inst_i</code> (buyruk) sinyali de 0 değerini alır. Bu durumda tüm kontrol sinyallerine 0 değeri atanır ve geçerli saat çevriminde işlemci içerisinde hiçbir eylem gerçekleşmez. Daha sonra bu yöntemi boru hattının ikinci aşamasını boşaltmak amacıyla kullanacağız.

### İşlemci Parçalarını Birleştirme

Matrak işlemcimizin (ilk bölümünün) tüm parçalarını ayrı ayrı tasarladık ve şimdi sıra bunları birleştirmeye geldi. Dolayısıyla burada yapacağımız iş oldukça basit, matrak isimli bir modül altında diğer modüllerin birbirleri ile bağlayacağız. Matrak işlemci modülünün kodu aşağıda verilmiştir.

```verilog
module matrak (
   input                clk_i,
   input                rst_i,
   input [31:0]         inst_i,     // Bellekten gelen buyruk
   output [31:0]        inst_addr_o // Belleğe giden buyruk adresi
);

   fetch f1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .pc_o(inst_addr_o)
   );

   // Boru hattı kaydedicisi bağlantıları
   wire [31:0] fd2d_inst;

   fd_regs fd1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .inst_f_i(inst_i),
      .inst_d_o(fd2d_inst)
   );

   // Çözme modülü bağlantıları
   wire c2d_regfile_wen;
   wire [2:0] c2d_imm_ext_sel;
   wire [31:0] w2d_result;
   wire [31:0] d2a_reg_a;
   wire [31:0] d2a_reg_b;
   wire [31:0] d2a_imm_ext;

   decode d1 (
      .clk_i(clk_i),
      .regfile_wen_i(c2d_regfile_wen),
      .imm_ext_sel_i(c2d_imm_ext_sel),
      .inst_i(fd2d_inst),
      .result_i(w2d_result),
      .reg_a_o(d2a_reg_a),
      .reg_b_o(d2a_reg_b),
      .imm_ext_o(d2a_imm_ext)
   );

   // ALU bağlantıları
   wire c2a_alu_sel;
   wire [3:0] c2a_alu_fun;
   wire [31:0] a2w_alu_out;

   alu a1 (
      .alu_sel_i(c2a_alu_sel),
      .alu_fun_i(c2a_alu_fun),
      .reg_a_i(d2a_reg_a),
      .reg_b_i(d2a_reg_b),
      .imm_ext_i(d2a_imm_ext),
      .alu_out_o(a2w_alu_out)
   );

   writeback w1 (
      .alu_out_i(a2w_alu_out),
      .result_o(w2d_result)
   );

   controller c1 (
      .inst_i(fd2d_inst),
      .regfile_wen_o(c2d_regfile_wen),
      .imm_ext_sel_o(c2d_imm_ext_sel),
      .alu_sel_o(c2a_alu_sel),
      .alu_fun_o(c2a_alu_fun)
   );

endmodule
```

<p align="center">
  <img src="/matrak.png"/>
</p>

<p align="center">
  Matrak işlemcimizin üst modül şeması
</p>

## İşlemci ve Belleği Bir Araya Getirme

Şimdi tasarladığımız bellek ve işlemci modüllerini bir araya getirip sistemi tamamlayacağız. İlk olarak işlemci modülümüzü oluşturan verilog kodlarını "matrak.v" dosyasında toplayalım.

matrak.v:

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Processor Module

module matrak (
   input                clk_i,
   input                rst_i,
   input [31:0]         inst_i,     // Bellekten gelen buyruk
   output [31:0]        inst_addr_o // Belleğe giden buyruk adresi
);

   fetch f1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .pc_o(inst_addr_o)
   );

   // Boru hattı kaydedicisi bağlantıları
   wire [31:0] fd2d_inst;

   fd_regs fd1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .inst_f_i(inst_i),
      .inst_d_o(fd2d_inst)
   );

   // Çözme modülü bağlantıları
   wire c2d_regfile_wen;
   wire [2:0] c2d_imm_ext_sel;
   wire [31:0] w2d_result;
   wire [31:0] d2a_reg_a;
   wire [31:0] d2a_reg_b;
   wire [31:0] d2a_imm_ext;

   decode d1 (
      .clk_i(clk_i),
      .regfile_wen_i(c2d_regfile_wen),
      .imm_ext_sel_i(c2d_imm_ext_sel),
      .inst_i(fd2d_inst),
      .result_i(w2d_result),
      .reg_a_o(d2a_reg_a),
      .reg_b_o(d2a_reg_b),
      .imm_ext_o(d2a_imm_ext)
   );

   // ALU bağlantıları
   wire c2a_alu_sel;
   wire [3:0] c2a_alu_fun;
   wire [31:0] a2w_alu_out;

   alu a1 (
      .alu_sel_i(c2a_alu_sel),
      .alu_fun_i(c2a_alu_fun),
      .reg_a_i(d2a_reg_a),
      .reg_b_i(d2a_reg_b),
      .imm_ext_i(d2a_imm_ext),
      .alu_out_o(a2w_alu_out)
   );

   writeback w1 (
      .alu_out_i(a2w_alu_out),
      .result_o(w2d_result)
   );

   controller c1 (
      .inst_i(fd2d_inst),
      .regfile_wen_o(c2d_regfile_wen),
      .imm_ext_sel_o(c2d_imm_ext_sel),
      .alu_sel_o(c2a_alu_sel),
      .alu_fun_o(c2a_alu_fun)
   );

endmodule

module fetch (
   input                clk_i,
   input                rst_i,
   output reg [31:0]    pc_o
);

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin // Sıfırlama sinyali geldiyse program sayacına başlangıç adresini ata
         pc_o <= 32'h0000_0000;
      end else begin // Saat sinyali geldiyse program sayacına 4 ekle
         pc_o <= pc_o + 4;
      end
   end

endmodule

module fd_regs (
   input                clk_i,
   input                rst_i,
   input [31:0]         inst_f_i, // Buyruk sinyali (bellekten geliyor)
   output reg [31:0]    inst_d_o  // Boru hattı kaydedicisinin çıkışı 
);

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         inst_d_o <= 32'b0;
      end else begin
         inst_d_o <= inst_f_i;
      end
   end

endmodule

module decode (
   input                   clk_i,
   input                   regfile_wen_i, // Kaydedici dosyası yazma yetkilendirme
   input [2:0]             imm_ext_sel_i, // İvedi genişletici format seçimi
   input [31:0]            inst_i,        // Boru hattı kaydedicisinden gelen buyruk
   input [31:0]            result_i,      // Hedef kaydedicisine (rd) yazılacak değer 
   output [31:0]           reg_a_o,       // Birinci kaynak kaydedicisinin (rs1) değeri
   output [31:0]           reg_b_o,       // İkinci kaynak kaydedicisinin (rs2) değeri
   output reg [31:0]       imm_ext_o      // İvedi genişleticinin çıkışı
);

   // 32 bit genişlikte 32 adet kaydedicili kaydedici dosyası
   reg [31:0] regfile [31:0];

   // Kaydedici adreslerini buyruktan ayıkla
   wire [4:0] reg_a_addr      = inst_i[19:15];  // rs1 adres
   wire [4:0] reg_b_addr      = inst_i[24:20];  // rs2 adres
   wire [4:0] target_reg_addr = inst_i[11:7];   // rd adres

   // Kaydedici dosyasından oku
   assign reg_a_o = (reg_a_addr == 5'b0) ? 32'b0 : regfile[reg_a_addr]; // rs1 değeri
   assign reg_b_o = (reg_b_addr == 5'b0) ? 32'b0 : regfile[reg_b_addr]; // rs2 değeri

   // Kaydedici dosyasına yaz
   always @(posedge clk_i) begin
      if (regfile_wen_i) begin
         regfile[target_reg_addr] <= result_i;
      end
   end

   // İvedi genişletici
   always @(*) begin
      case (imm_ext_sel_i)
         3'b000   : imm_ext_o = {{20{inst_i[31]}}, inst_i[31:20]};
         default  : imm_ext_o = 32'b0; 
      endcase
   end

endmodule

module alu (
   input                      alu_sel_i,  // İkinci işlenenin seçim sinyali (rs2 veya imm)
   input [3:0]                alu_fun_i,  // İşlem seçim sinyali
   input [31:0]               reg_a_i,    // rs1 değeri
   input [31:0]               reg_b_i,    // rs2 değeri
   input [31:0]               imm_ext_i,  // imm değeri
   output reg [31:0]          alu_out_o   // Sonuç değeri
);

   // Birinci işlenen iki buyruk formatında da sabit.
   wire signed [31:0] alu_a = reg_a_i;
   // İkinci işlenen seçim sinyaline göre belirleniyor.
   wire signed [31:0] alu_b = alu_sel_i ? imm_ext_i : reg_b_i;

   always @(*) begin
      case (alu_fun_i)
         4'b0000  : alu_out_o = alu_a + alu_b;  // Toplama 
         4'b0001  : alu_out_o = alu_a - alu_b;  // Çıkarma
         4'b0010  : alu_out_o = alu_a & alu_b;  // VE
         4'b0011  : alu_out_o = alu_a ^ alu_b;  // XOR
         4'b0100  : alu_out_o = alu_a | alu_b;  // VEYA
         default  : alu_out_o = 32'bx;          // Geçersiz alu_fun_i sinyali
      endcase
   end

endmodule

module writeback (
   input  [31:0]              alu_out_i,
   output [31:0]              result_o
);

   assign result_o = alu_out_i;
   
endmodule

module controller (
   input [31:0]               inst_i,        // Boru hattı kaydedicisinden gelen buyruk
   output                     regfile_wen_o, // Kaydedici dosyası yazma yetkilendirme sinyali
   output [2:0]               imm_ext_sel_o, // İvedi genişletici format seçim sinyali
   output                     alu_sel_o,     // ALU ikinci işlenen seçim sinyali
   output reg [3:0]           alu_fun_o      // ALU işlem seçim sinyali
);

   // Buyruğun gerekli bölümleri ayıklanıyor.
   wire [6:0] opcode = inst_i[6:0];
   wire [2:0] funct3 = inst_i[14:12];
   wire [6:0] funct7 = inst_i[31:25];

   wire [1:0] alu_dec;

   reg [6:0] control_signals;
   assign {regfile_wen_o, imm_ext_sel_o, alu_sel_o, alu_dec} = control_signals;

   always @(*) begin
      case (opcode)  // Opcode'a göre kontrol sinyallerinin değerleri belirleniyor. 
         7'b0110011  : control_signals = 7'b1_xxx_0_11; // R-type buyruk
         7'b0010011  : control_signals = 7'b1_000_1_11; // I-type buyruk
         7'b0000000  : control_signals = 7'b0_000_0_00; // Sıfırlama durumu
         default     : control_signals = 7'bx_xxx_x_xx; // Geçersiz buyruk
      endcase
   end

   // Buyruk R-type ise ve funct7 değeri 0x20 ise çıkarma işlemi anlamına gelir.
   wire sub = opcode[5] & funct7[5];

   // ALU'da yapılacak işlem belirleniyor.
   always @(*) begin
      case (alu_dec)
         2'b11    : // R-type veya I-type
            case (funct3)
               3'b000   : // add-addi veya sub buyruğu
                  if (sub) begin
                     alu_fun_o = 4'b0001; // sub
                  end else begin
                     alu_fun_o = 4'b0000; // add, addi
                  end
               3'b100   : alu_fun_o = 4'b0011; // xor, xori
               3'b110   : alu_fun_o = 4'b0100; // or, ori
               3'b111   : alu_fun_o = 4'b0010; // and, andi
               default  : alu_fun_o = 4'b0000;
            endcase
         default  : alu_fun_o = 4'b0000; // Varsayılan işlemi toplama olarak belirledik.
      endcase
   end

endmodule
```

Şimdi de yazının en başında hazırladığımız bellek modülünü hatırlayalım.

memory.v:

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Main Memory Module

module memory (
   input          clk_i,
   input          wen_i,
   input [31:0]   addr_i,
   input [31:0]   data_i,
   input [31:0]   data_o
);

   // 512x32 bit = 16384 bit = 2048 bayt = 2 kibibayt (2 KiB)
   reg [31:0] mem [511:0];

   // Programı belleğe yükle
   initial begin
      $readmemh("program.mem", mem);
   end

   // Asenkron okuma
   assign data_o = mem[addr_i[31:2]];

   // Senkron yazma
   always @(posedge clk_i) begin
      if (wen_i) begin
         mem[addr_i[31:2]] <= data_i;
      end
   end

endmodule
```

> :pushpin: Bellek modülümüze program yüklemek için <code>$readmemh("program.mem", mem);</code> satırını ekledik. program.mem dosyasının detaylarına benzetim kısmında değineceğiz.

Şimdi belleği ve işlemciyi birbirine bağlayan üst modüle geçelim. Bu modül hiyerarşinin en üstünde konumlandığı için "top.v" olarak isimlendirdik.

top.v:

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Top Module

module top (
   input             clk_i,
   input             rst_i
);

   wire [31:0] inst;
   wire [31:0] inst_addr;

   matrak mt1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .inst_i(inst),
      .inst_addr_o(inst_addr)
   );

   memory me1 (
      .clk_i(clk_i),
      .wen_i(1'b0),
      .addr_i(inst_addr),
      .data_i(32'b0),
      .data_o(inst)
   );

endmodule
```

> :pushpin: Biz belleği RAM yapısında, yani yazılıp okunabilir biçimde tasarlamıştık. Fakat şimdilik sadece bellekten okuma ile ilgileneceğimiz için <code>wen_i</code> ve <code>data_i</code> sinyallerini sıfıra çekiyoruz.

## Benzetim

İyi haber şu ki birkaç buyruk içeren işlemcimizi tasarladık. Ama işimiz burada bitmedi çünkü henüz işlemciyi çalışırken görmedik. Peki işlemcimizi en kolay ve en ucuz şekilde nasıl çalıştırabiliriz? Cevap basit: bilgisayar ortamında benzetim.

İşlemcimizi test edebileceğimiz çeşitli açık kaynak ve ticari yazılımlar mevcut. Biz benzetim yazılımı tercihimizi, Xilinx firmasının FPGA ve türevi ürünleri için sağladığı geliştirme ortamı olan Vivado'dan yana kullanacağız.

Tasarladığımız sistemi test etmek için "top_tb.v" isimli basit bir testbench modülü yazıyoruz. Bu modül sisteme saat sinyali sağlıyor ve başlangıçta bir sıfırlama sinyali gönderiyor.

top_tb.v:

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Top Module Testbench

module top_tb ();

   reg tb_clk_i;
   reg tb_rst_i;

   top t1 (
      .clk_i(tb_clk_i),
      .rst_i(tb_rst_i)
   );

   initial begin
      tb_clk_i = 1'b0;
      tb_rst_i = 1'b0;
      #1 tb_rst_i = 1'b1;
      #1 tb_rst_i = 1'b0;
      forever begin
         #1 tb_clk_i = ~tb_clk_i;
      end
   end

endmodule
```

Donanımımız hazır ama işlemcimizin çalıştıracağı test programını da hazırlamalıyız. RISC-V assembly ile işlemciye eklediğimiz buyrukların çalışmasını doğrulayacak basit bir test kodu yazıyoruz:

```
# Assembly de yorum satırı için "#" kullanılır.
# Yorum kısımlarında ilgili buyruğun sonucu verilmiştir.

.global _start
.text

_start: 
   addi  x5, x0, 4      # x5 = 4
   addi  x6, x0, 8      # x6 = 8
   add   x7, x5, x6     # x7 = 12
   sub   x7, x7, x5     # x7 = 8
   xor   x7, x7, x6     # x7 = 0
   ori   x5, x6, 153    # x5 = 153
   andi  x7, x5, 255    # x7 = 153
```

Yazdığımız assembly kodu belleğe gömülmeden önce makine koduna dönüştürülmeli. Assembly kodunu makine koduna dönüştüren  programlara "assembler" denir. Assembler programı Toolchain denilen derleyici araçlarının toplandığı bir pakette bulunur. İlerleyen bölümlerde derleyici araçlarınının kurulumuna değineceğiz, fakat şimdilik işi basit tutmak için [internetten çalışan bir assembler](https://riscvasm.lucasteske.dev/) kullanmamızda sakınca yok. Adını "program.mem" olarak değiştirdiğimiz bir dosya oluşturup, ürettirdiğimiz makine kodunu içerisine kopyalıyoruz. Assembly test kodumuzun onaltılık tabanda makine koduna çevrilmiş hali aşağıda verilmiştir.

program.mem:

```
00400293
00800313
006283b3
405383b3
0063c3b3
09936293
0ff2f393
```

Vivado'yu açıp yeni bir proje oluşturuyoruz, daha sonra matrak.v, memory.v, top.v, top_tb.v, program.mem dosyalarını oluşturduğumuz projeye ekliyoruz. Şimdilik yalnızca benzetim yapacağımız için özel bir kart seçmeye yahut "constraint" dosyası eklemeye ihtiyacımız yok. Daha sonra Simulation kısmından Run Behavioral Simulation seçeneğine tıklıyoruz. Ardından decode modülü altında yer alan "regfile" isimli kaydedici dosyamıza sağ tıklayıp Add to Wave Window diyoruz. Relaunch Simulation tuşuna basarak kaydedici dosyamızı inceliyoruz.

<p align="center">
  <img src="/mvivado1.png"/>
</p>

> :pushpin: Vivado benzetim ortamında, sinyal değerleri varsayılan olarak onaltılık tabanda görüntüleniyor. Dilerseniz dalgaformu ekranındaki sinyale sağ tıklayıp tabanını değiştirebilirsiniz.

Ve sonuç tam da beklediğimiz gibi.

Matrak işlemcimiz henüz dört başı mamur olmaktan çok uzak ancak artık tarafımızı seçmiş bulunuyoruz.

<p align="center">
  <img src="/x86vsarm.png"/>
</p>

Bu yazılık bu kadar\.\.\.

> :scroll: Bu bölümün kodlarına erişmek için [tıklayın](https://github.com/necaticakaci/matrak/tree/main/episodes/ep1).
