---
title: "RISC-V İşlemci Tasarımı - Bölüm 2: Dallanma ve Kaydırma Buyrukları"
date: 2023-10-06
description: Bu bölümde işlemcimizin eksik R-type ve I-type buyruklarını tamamlıyor ve dallanma buyruklarını ekliyoruz.
math: true
---

Bir önceki yazıda Matrak işlemcimiz R-type ve I-type formatında birkaç buyruğu çalıştırabilir hale gelmişti. Şimdi ise RV32I buyruk kümesinin geri kalan R-type ve I-type buyrukları ile B-type dallanma buyruklarını ekleyeceğiz. Dallanma buyruklarında da belirli bir koşulun gerçekleşip gerçekleşmeyeceğine bakacağız ve gerekirse program sayacında tutulan adresi değiştireceğiz. İşlemcimizin bu buyrukları desteklemesi için halihazırda tasarlamış olduğumuz birimlerde bazı değişiklikler yapmamız gerekecek.

## Eklenecek R-type ve I-type buyruklar

R-type ve I-type buyruk formatına tekrar bir göz atalım:

| Bitler  | 31 - 25 | 24 - 20 | 19 - 15 | 14 - 12 | 11 - 7  | 6 - 0   |
| ------- | ------- | ------- | ------- | ------- | ------- | ------- |
| R-type  | funct7  | rs2     | rs1     | funct3  | rd      | opcode  |
| I-type  | imm[11:0]        || rs1     | funct3  | rd      | opcode  |

R-type ve I-type buyrukları hatırlayalım:

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

<code>add, addi, sub, xor, xori, or, ori, and, andi</code> buyruklarını bir önceki bölümde zaten işlemcimize eklemiştik. Geriye kalan buyruklara kısaca değinelim:

<code>sll, slli</code>: Shift left logical, Shift left logical immediate (Mantıksal sola kaydırma)

<code>srl, srli</code>: Shift right logical, Shift ight logical immediate (Mantıksal sağa kaydırma)

<code>sra, srai</code>: Shift right arithmetic, Shift right arithmetic immediate (Aritmetik sağa kaydırma)

<code>slt, slti</code>: Set less than, Set less than immediate (Küçükse bir yap)

<code>sltu, sltiu</code>: Set less than unsigned, Set less than immediate unsigned (İşaretsiz sayılar için küçükse bir yap)

> :bulb: Kaydırma buyrukları çarpma ve bölme işlemi yerine kullanılabilir. Bir sayıyı n bit sola kaydırmak sayıyı 2<sup>n</sup> ile çarparken yine aynı şekilde bir sayıyı n bit sağa kaydırmak 2<sup>n</sup> değerine böler.

> :pushpin: Aritmetik ve mantıksal olmak üzere iki tür sağa kaydırma işlemi vardır. Mantıksal kaydırmada boşluklar 0 ile doldurulurken aritmetik kaydırmada sayının en anlamlı bitiyle doldurulur. Dolayısıyla aritmetik kaydırmada sayının işareti korunmuş olur.

> :pushpin: I-type buyrukların funct7 değerleri yoktur. Ancak srli ve srai kaydırma buyruklarının funct3 değerleri eşit ve istisna olarak funct7 değerine sahipler. Kaydırma buyrukları ivedi değerin yalnızca 5 bitine ihtiyaç duyduğu için funct7 alanı kullanılabilir oluyor.

## Dallanma Buyrukları

Bir dallanma buyruğu işleme alındığında, dallanma koşulu sağlanmış ise program sayacındaki adresi, buyruktaki ivedi değerle toplayıp tekrar program sayacına yazmak gerekir. Bu sayede dallanma gerçekleştirilmiş olur ve program, normal akışı (PC + 4) yerine başka bir adresten devam eder.

```
start:   # programın buradan başladığını farzedelim.
   addi  x5, x0, 4      # x5 = 4
   addi  x6, x0, 8      # x6 = 8
   bne   x5, x6, main   # x5 ve x6 nın değeri eşit değilse main etiketine dallan
   add   x5, x5, x5     # Yukarıda dallanma gerçekleşeceği için bu buyruk işleme alınmaz.

main:    # program buradan devam eder.
   xor   x7, x7, x6
   ori   x5, x6, 153
   andi  x7, x5, 255
```

> :pushpin: Yüksek seviyeli dilde yazılmış kod (Örn. C) makine koduna çevrilirken, kodda yer alan "if" ve "else" gibi yapılar dallanma buyruklarına dönüştürülür.

Dallanma buyruklarının formatı B-type olarak belirlenmiş:

| Bitler  | 31 - 25        | 24 - 20 | 19 - 15 | 14 - 12 | 11 - 7       | 6 - 0   |
| ------- | -------------- | ------- | ------- | ------- | ------------ | ------- |
| B-type  | imm[12\|10:5]  | rs2     | rs1     | funct3  | imm[4:1\|11] | opcode  |

> :bulb: B-type formatının ivedi kısımları biraz karmaşık görünüyor. Bu henüz karşılaşmadığımız S-type formatı ile bit alanlarını eşlemek için yapılmış akıllıca bir numara. Aynı amaçla kullanılan bit alanlarının donanımı basitleştirdiğine daha önce değinmiştik.

> :pushpin: B-type buyrukların hedef kaydedicisi yok, dolayısıyla işlem sonucunda kaydedici dosyası güncellenmeyecek.

**B-type Buyruklar:**

| Buyruk          | Tanım                             | opcode          | funct3          |
| --------------- | --------------------------------- | --------------- | --------------- |
| beq             | if (rs1 == rs2)    PC = PC + imm  | 1100011         | 0x0             |
| bne             | if (rs1 != rs2)    PC = PC + imm  | 1100011         | 0x1             |
| blt             | if (rs1 < rs2)     PC = PC + imm  | 1100011         | 0x4             |
| bge             | if (rs1 >= rs2)    PC = PC + imm  | 1100011         | 0x5             |
| bltu            | if (u(rs1 < rs2))  PC = PC + imm  | 1100011         | 0x6             |
| bgeu            | if (u(rs1 < rs2))  PC = PC + imm  | 1100011         | 0x7             |

<code>beq</code>: Branch equal (Eşitse dallan)

<code>bne</code>: Branch not equal (Eşit değilse dallan)

<code>blt</code>: Branch less than (Küçükse dallan)

<code>bge</code>: Branch greater or equal (Büyük eşitse dallan)

<code>bltu</code>: Branch less than unsigned (İşaretsiz sayılar için küçükse dallan)

<code>bgeu</code>: Branch greater or equal unsigned (İşaretsiz sayılar için büyük eşitse dallan)

> :bulb: Boru hattına sahip işlemcilerde, dallanma buyruğu yürütme aşamasına gelmeden yeni buyruk okunması gerekebilir. Bu noktada dallanma sonucu belirsizdir, dolayısıyla hangi adresten okuma yapılacağı bilinemez. Bu soruna çözüm olarak dallanma tahmini algoritmaları geliştirilmiştir. Dallanma tahmini yapan işlemcilerde yapılan tahmin doğru ise işlemci normal çalışmasına devam eder, eğer tahmin yanlış ise hatalı buyruğun boru hattından çıkartılması gerekir. Boru hattının boşaltılması işlemciyi yavaşlattığı için yüksek tutarlılıkta çalışabilen bir dallanma tahmin birimi, işlemci başarımını arttıracaktır.

> :pushpin: Matrak işlemcimizi getirme ve yürütme olmak üzere 2 aşamalı boru hattına sahip olacak şekilde tasarladık. Getirme birimi dallanmanın gerçekleşmeyeceğini umarak adrese 4 ekler. Dallanma buyruğu ikinci aşamaya geçtiğinde eğer dallanma koşulu sağlandıysa getirme birimindeki program sayacı güncellenir. Bu esnada boru hattı kaydedicisinin girişinde bir sonraki (PC+4) buyruk bulunur. Hatalı yürütmeyi engellemek için boru hattı kaydedicisinin sıfırlanması gerekir. Bu sayede bir sonraki saat çevriminde boru hattı kaydedicisinin çıkışında sıfır değeri görülür ve bu bizim işlemcimiz tarafından "nop" yani işlem yok olarak algılanır. Özetle bir dallanma buyruğu işleme alınır ve dallanma gerçekleşirse bir çevrim boşa gider. Bir sonraki çevrimde işlemci ilgili adrese dallanmış olur normal çalışmasına devam eder.

## Getirme Birimi

Yine getirme birimi ile başlıyoruz. Bir önceki yazıda hazırlamış olduğumuz getirme birimi, yalnızca geçerli adrese çevrim başına 4 ekliyordu. Dallanma buyruklarını desteklemek için 32-bit genişlikte harici adres girişine ve bir adet seçim girişine ihtiyacımız var. Seçim girişi kontrol birimi tarafından yönetilecek. Dallanma adresi de bu yazıda tasarlayacağımız "adres hesaplayıcı" biriminden getirme birimine iletilecek.

<code>pc_sel_i</code> değeri 1 ise program sayacına <code>pc_ext_i</code> adresi yüklenir.

```verilog
module fetch (
   input                clk_i,
   input                rst_i,
   input                pc_sel_i,   // Program sayacı seçim girişi
   input [31:0]         pc_ext_i,   // Dallanma adres girişi
   output reg [31:0]    pc_o        // Program sayacı çıkışı
);

   // Program sayacına 4 ekle
   wire [31:0] pc_plus = pc_o + 4;

   // Dallanma adresi veya PC + 4 
   wire [31:0] pc_next = pc_sel_i ? pc_ext_i : pc_plus;

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         pc_o <= 32'h0000_0000;
      end else begin
         pc_o <= pc_next;
      end
   end

endmodule
```

## Boru Hattı Kaydedicileri

Önceki bölümde buyruğu saklayan bir adet 32-bitlik boru hattı kaydedicimiz vardı. Dallanma buyruklarını işleyebilmek için program sayacının değerine ihtiyaç duyuyoruz, bu sebeple getirme biriminden gelen program sayacı değerini saklayacak 32-bitlik bir boru hattı kaydedicisi daha ekliyoruz. Dallanma koşulu gerçekleştiğinde <code>inst_f_i</code> sinyali yanlış buyruğu tutuyor olacağı için geçerli çevrimde boru hattının boşaltılması yani kaydedicilerin sıfırlanması gerekecek. Bu işlemi kontrol birimi üzerinden <code>clear_i</code> sinyali vasıtasıyla yapacağız.

```verilog
module fd_regs (
   input                clk_i,
   input                rst_i,
   input                clear_i,    // Sıfırlama sinyali (boru hattı boşaltma)
   input [31:0]         inst_f_i,   // Buyruk girişi (bellekten geliyor)
   input [31:0]         pc_f_i,     // Progam sayacı girişi (getirme biriminden geliyor)
   output reg [31:0]    inst_d_o,   // Buyruk çıkışı (yürütme aşamasına gidiyor)
   output reg [31:0]    pc_d_o      // Program sayacı çıkışı (yürütme aşamasına gidiyor)
);

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         inst_d_o <= 32'b0;
         pc_d_o   <= 32'b0;
      end else begin
         if (clear_i) begin // Boru hattı boşaltılıyor.
            inst_d_o <= 32'b0;
            pc_d_o   <= 32'b0;
         end else begin
            inst_d_o <= inst_f_i;
            pc_d_o   <= pc_f_i;
         end
      end
   end

endmodule
```

## Çözme Birimi

Çözme biriminde yapacağımız değişiklik yalnızca ivedi genişleticiye B-type formatının eklenmesi olacak.

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
         3'b000   : imm_ext_o = {{20{inst_i[31]}}, inst_i[31:20]}; // I-type
         3'b001   : imm_ext_o = {{20{inst_i[31]}}, inst_i[7], inst_i[30:25], inst_i[11:8], 1'b0}; // B-type
         default  : imm_ext_o = 32'b0; 
      endcase
   end

endmodule
```

## ALU

Kaydırma işlemlerini ilgili operatörleri kullanarak kolayca gerçekleştirebiliyoruz. 32-bit sayılarla kaydırma işlemi yaptığımız için ikinci işlenen sayının 5-bit olması yeterli.

Hatırlayacağınız üzere dallanma koşulları büyük mü, küçük mü, eşit mi, eşit değil mi gibi işlemlerden meydana geliyordu. Her koşul için ayrı bir işlem yapmamıza gerek yok zira bazı ifadeler birbirinin zıddı oluyor.

<code>alu_a == alu_b</code>: Sonuç 1 ise <code>alu_a = alu_b</code>, Sonuç 0 ise <code>alu_a != alu_b</code>

<code>alu_a < alu_b</code>: Sonuç 1 ise <code>alu_a < alu_b</code>, Sonuç 0 ise <code>alu_a >= alu_b</code>

ALU sonucunun sıfır olduğunu belirten <code>alu_zero_o</code> sinyalini tanımlıyoruz. Bu sinyal kontrol birimine bağlanacak ve dallanma buyruğu işlenirken, koşulun sağlanıp sağlanmadığı bu sinyal vasıtasıyla anlaşılacak.

<code>slt, slti, sltu, sltiu</code> buyrukları ile <code>blt ve bltu</code> dallanma buyrukları ortak işlemler gerektirdiği için başka bir ifade eklemeye ihtiyaç duymuyoruz. Dallanma buyrukları ile aralarındaki fark, sonucun kaydedici dosyasına yazılması olacak.

```verilog
module alu (
   input                      alu_sel_i,  // İkinci işlenenin seçim sinyali (rs2 veya imm)
   input [3:0]                alu_fun_i,  // İşlem seçim sinyali
   input [31:0]               reg_a_i,    // rs1 değeri
   input [31:0]               reg_b_i,    // rs2 değeri
   input [31:0]               imm_ext_i,  // imm değeri
   output                     alu_zero_o, // Sonuç sıfır sinyali
   output reg [31:0]          alu_out_o   // Sonuç değeri
);

   // Birinci işlenen iki buyruk formatında da sabit.
   wire signed [31:0] alu_a = reg_a_i;
   // İkinci işlenen seçim sinyaline göre belirleniyor.
   wire signed [31:0] alu_b = alu_sel_i ? imm_ext_i : reg_b_i;

   // Sonuç 0'a eşit ise alu_zero_o sinyali 1 olur.
   assign alu_zero_o = ~(|alu_out_o);

   always @(*) begin
      case (alu_fun_i)
         4'b0000  : alu_out_o = alu_a + alu_b;           // Toplama 
         4'b0001  : alu_out_o = alu_a - alu_b;           // Çıkarma
         4'b0010  : alu_out_o = alu_a & alu_b;           // VE
         4'b0011  : alu_out_o = alu_a ^ alu_b;           // XOR
         4'b0100  : alu_out_o = alu_a | alu_b;           // VEYA
         4'b0101  : alu_out_o = alu_a << alu_b[4:0];     // Sola kaydırma
         4'b0110  : alu_out_o = alu_a >> alu_b[4:0];     // Sağa kaydırma
         4'b0111  : alu_out_o = alu_a >>> alu_b[4:0];    // Aritmetik sağa kaydırma
         4'b1000  : alu_out_o = {31'b0, alu_a == alu_b}; // Eşitse alu_out_o = 1, değilse alu_out_o = 0 (beq, bne)
         4'b1001  : alu_out_o = {31'b0, alu_a < alu_b};  // Küçükse alu_out_o = 1, değilse alu_out_o = 0 (blt, bge, slt, slti)
         4'b1010  : alu_out_o = {31'b0, $unsigned(alu_a) < $unsigned(alu_b)}; // (İşaretsiz) küçükse alu_out_o = 1, değilse alu_out_o = 0 (bltu, bgeu, sltu, sltiu)
         default  : alu_out_o = 32'bx;                   // Geçersiz alu_fun_i sinyali
      endcase
   end

endmodule
```

## Adres Hesaplayıcı

Dallanma koşulunun sağlanıp sağlanmadığının hesabı ALU'da halledilecek, bu tamam. Fakat dallanma adresini de hesaplamalıyız. Dallanma adresi, program sayacının değeriyle buyrukta yer alan ivedi değerin toplamı (PC = PC + imm) olduğunu biliyoruz. Bunun için iki değeri toplayan basit bir modül tasarlıyoruz.

```verilog
module address_calculator (
   input [31:0]               pc_i,       // Boru hattı kaydedicisinden gelen program sayacının değeri
   input [31:0]               imm_ext_i,  // Çözme biriminden gelen ivedi değer
   output [31:0]              pc_ext_o    // Program sayacına yazılacak adres
);

   assign pc_ext_o = pc_i + imm_ext_i;

endmodule
```

## Geriyazma Birimi

Geriyazma biriminde şimdilik herhangi bir değişiklik yapmayacağız. Bir önceki bölümde hazırladığımız modülü kullanmaya devam ediyoruz.

```verilog
module writeback (
   input  [31:0]              alu_out_i,
   output [31:0]              result_o
);

   assign result_o = alu_out_i;

endmodule
```

## Kontrol Birimi

Yeni eklediğimiz buyrukları desteklemek için kontrol biriminde çeşitli değişiklikler yapmalıyız. Önceki bölümde, kontrol birimimiz R-type ve I-type buyrukları işleyebilir hale gelmişti. Kaydırma ve küçükse bir yap buyrukları için sadece ALU fonksiyon çözücüsüne eklemeler yapıyoruz.

Dallanma buyrukları için opcode çözücüsüne B-type formatını eklemeliyiz. Ayrıca bu bölümde <code>branch_op</code> adında bir kontrol sinyali daha tanımladık. Bu sinyalin görevi işlenen buyruğun dallanma buyruğu olduğunu belirtmek. <code>branch_op</code> sinyalinin 1 olduğu durumlarda ALU'daki işlem sonucuna göre dallanma kararı verilecek.

Bir dallanma (B-type) buyruğu geldiğinde kontrol sinyalleri şöyle değişir:

* <code>regfile_wen_o</code>: 0 (kaydedici dosyası güncellenmeyecek)
* <code>imm_ext_sel_o</code>: 001 (ivedi türü B-type olarak seçilecek)
* <code>alu_sel_o</code>: 0 (ALU'da işlemler rs1 ve rs2 arasında gerçekleşecek)
* <code>branch_op</code>: 1 (işlenen bir dallanma buyruğu)

```verilog
module controller (
   input [31:0]               inst_i,        // Boru hattı kaydedicisinden gelen buyruk
   input                      alu_zero_i,    // ALU'dan gelen sonuç sıfır sinyali
   output                     regfile_wen_o, // Kaydedici dosyası yazma yetkilendirme sinyali
   output [2:0]               imm_ext_sel_o, // İvedi genişletici format seçim sinyali
   output                     alu_sel_o,     // ALU ikinci işlenen seçim sinyali
   output reg [3:0]           alu_fun_o,     // ALU işlem seçim sinyali
   output                     pc_sel_o,      // Program sayacı adres seçim sinyali
   output                     clear_o        // Boru hattı boşaltma sinyali
);

   // Buyruğun gerekli bölümleri ayıklanıyor.
   wire [6:0] opcode = inst_i[6:0];
   wire [2:0] funct3 = inst_i[14:12];
   wire [6:0] funct7 = inst_i[31:25];

   wire [1:0] alu_dec;
   wire branch_op;

   reg [7:0] control_signals;
   assign {regfile_wen_o, imm_ext_sel_o, alu_sel_o, alu_dec, branch_op} = control_signals;

   always @(*) begin
      case (opcode)
         7'b0110011  : control_signals = 8'b1_xxx_0_11_0; // R-type buyruk
         7'b0010011  : control_signals = 8'b1_000_1_11_0; // I-type buyruk
         7'b1100011  : control_signals = 8'b0_001_0_01_1; // B-type buyruk
         7'b0000000  : control_signals = 8'b0_000_0_00_0; // Sıfırlama durumu
         default     : control_signals = 8'bx_xxx_x_xx_x; // Geçersiz buyruk
      endcase
   end

   // Buyruk R-type ise ve funct7 değeri 0x20 ise çıkarma işlemi anlamına gelir.
   wire sub = opcode[5] & funct7[5];

   // ALU'da yapılacak işlem belirleniyor.
   always @(*) begin
      case (alu_dec)
         2'b01    : // B-type
            case (funct3)
               3'b000   : alu_fun_o = 4'b1000; // beq
               3'b001   : alu_fun_o = 4'b1000; // bne
               3'b100   : alu_fun_o = 4'b1001; // blt
               3'b101   : alu_fun_o = 4'b1001; // bge
               3'b110   : alu_fun_o = 4'b1010; // bltu
               3'b111   : alu_fun_o = 4'b1010; // bgeu
               default  : alu_fun_o = 4'bx;
            endcase
         2'b11    : // R-type veya I-type
            case (funct3)
               3'b000   : // add-addi veya sub buyruğu
                  if (sub) begin
                     alu_fun_o = 4'b0001; // sub
                  end else begin
                     alu_fun_o = 4'b0000; // add, addi
                  end
               3'b001   : alu_fun_o = 4'b0101; // sll, slli
               3'b010   : alu_fun_o = 4'b1001; // slt, slti
               3'b011   : alu_fun_o = 4'b1010; // sltu, sltiu
               3'b100   : alu_fun_o = 4'b0011; // xor, xori
               3'b101   : // srl, srli, sra, srai
                  if (funct7[5]) begin
                     alu_fun_o = 4'b0111; // sra, srai
                  end else begin
                     alu_fun_o = 4'b0110; // srl, srli
                  end
               3'b110   : alu_fun_o = 4'b0100; // or, ori
               3'b111   : alu_fun_o = 4'b0010; // and, andi
               default  : alu_fun_o = 4'b0000;
            endcase
         default  : alu_fun_o = 4'b0000; // Varsayılan işlem toplama
      endcase
   end

   reg branch_valid;

   always @(*) begin
      case (funct3)
         3'b000   : branch_valid = !alu_zero_i;   // beq
         3'b001   : branch_valid = alu_zero_i;    // bne
         3'b100   : branch_valid = !alu_zero_i;   // blt
         3'b101   : branch_valid = alu_zero_i;    // bge
         3'b110   : branch_valid = !alu_zero_i;   // bltu
         3'b111   : branch_valid = alu_zero_i;    // bgeu
         default  : branch_valid = 1'b0;
      endcase
   end

   assign pc_sel_o   = branch_op & branch_valid; // Dallanma durumu kontrol ediliyor.
   assign clear_o    = pc_sel_o; // Boru hattını boşalt

endmodule

```

Buyrukta tanımlı dallanma koşulu sağlandığında <code>branch_valid</code> sinyalinin değeri 1 olur. <code>branch_valid</code> ve <code>branch_op</code> sinyallerinin değeri 1 ise, <code>pc_sel_o</code> sinyali vasıtasıyla dallanma adresi program sayacına yüklenir ve boru hattı temizlenir. Böylelikle dallanma gerçekleşmiş olur.

Neredeyse bitti! Fakat sizin de gözünüze bir şeyler takıldı mı? Dallanma koşulunun gerçekleşip gerçekleşmediğini anlayabilmek amacıyla kurduğumuz case yapısında bir örüntü oluştu.

```verilog
   always @(*) begin
      case (funct3)
         3'b000   : branch_valid = !alu_zero_i;   // beq
         3'b001   : branch_valid = alu_zero_i;    // bne
         3'b100   : branch_valid = !alu_zero_i;   // blt
         3'b101   : branch_valid = alu_zero_i;    // bge
         3'b110   : branch_valid = !alu_zero_i;   // bltu
         3'b111   : branch_valid = alu_zero_i;    // bgeu
         default  : branch_valid = 1'b0;
      endcase
   end
```

<code>funct3</code> sinyalinin en anlamsız biti (LSB) 1 olduğu durumda <code>alu_zero_i</code> sinyalini aktarmamız gerekirken, 0 olduğu durumda <code>alu_zero_i</code> sinyalinin değilini aktarmamız gerekiyor. Öyleyse bu ifadeyi aşağıdaki gibi sadeleştirebiliriz.

```verilog
   assign branch_valid = funct3[0] ? alu_zero_i : !alu_zero_i;
```

## Tüm Parçaları Birleştirme

Şimdi çekirdeğin tüm modüllerini bir araya getirebiliriz. matrak.v dosyamızı aşağıdaki gibi güncelleyelim.

<details>
<summary>matrak.v: <mark>kodu göstermek için tıklayın</mark></summary>

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

   // Getirme birimi bağlantıları
   wire c2f_pc_sel;
   wire [31:0] ac2f_pc_ext;

   fetch f1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .pc_sel_i(c2f_pc_sel),
      .pc_ext_i(ac2f_pc_ext),
      .pc_o(inst_addr_o)
   );

   // Boru hattı kaydedicisi bağlantıları
   wire [31:0] fd2d_inst;
   wire [31:0] fd2ac_pc_d;
   wire c2fd_clear;

   fd_regs fd1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .clear_i(c2fd_clear),
      .inst_f_i(inst_i),
      .pc_f_i(inst_addr_o),
      .inst_d_o(fd2d_inst),
      .pc_d_o(fd2ac_pc_d)
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
   wire a2c_alu_zero;

   alu a1 (
      .alu_sel_i(c2a_alu_sel),
      .alu_fun_i(c2a_alu_fun),
      .reg_a_i(d2a_reg_a),
      .reg_b_i(d2a_reg_b),
      .imm_ext_i(d2a_imm_ext),
      .alu_zero_o(a2c_alu_zero),
      .alu_out_o(a2w_alu_out)
   );

   address_calculator ac1(
      .pc_i(fd2ac_pc_d),
      .imm_ext_i(d2a_imm_ext),
      .pc_ext_o(ac2f_pc_ext)
   );

   writeback w1 (
      .alu_out_i(a2w_alu_out),
      .result_o(w2d_result)
   );

   controller c1 (
      .inst_i(fd2d_inst),
      .alu_zero_i(a2c_alu_zero),
      .regfile_wen_o(c2d_regfile_wen),
      .imm_ext_sel_o(c2d_imm_ext_sel),
      .alu_sel_o(c2a_alu_sel),
      .alu_fun_o(c2a_alu_fun),
      .pc_sel_o(c2f_pc_sel),
      .clear_o(c2fd_clear)
   );

endmodule

module fetch (
   input                clk_i,
   input                rst_i,
   input                pc_sel_i,   // Program sayacı seçim girişi
   input [31:0]         pc_ext_i,   // Dallanma adres girişi
   output reg [31:0]    pc_o        // Program sayacı çıkışı
);

   // Program sayacına 4 ekle
   wire [31:0] pc_plus = pc_o + 4;

   // Dallanma adresi veya PC + 4 
   wire [31:0] pc_next = pc_sel_i ? pc_ext_i : pc_plus;

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         pc_o <= 32'h0000_0000;
      end else begin
         pc_o <= pc_next;
      end
   end

endmodule

module fd_regs (
   input                clk_i,
   input                rst_i,
   input                clear_i,    // Sıfırlama sinyali (boru hattı boşaltma)
   input [31:0]         inst_f_i,   // Buyruk girişi (bellekten geliyor)
   input [31:0]         pc_f_i,     // Progam sayacı girişi (getirme biriminden geliyor)
   output reg [31:0]    inst_d_o,   // Buyruk çıkışı (yürütme aşamasına gidiyor)
   output reg [31:0]    pc_d_o      // Program sayacı çıkışı (yürütme aşamasına gidiyor)
);

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         inst_d_o <= 32'b0;
         pc_d_o   <= 32'b0;
      end else begin
         if (clear_i) begin // Boru hattı boşaltılıyor.
            inst_d_o <= 32'b0;
            pc_d_o   <= 32'b0;
         end else begin
            inst_d_o <= inst_f_i;
            pc_d_o   <= pc_f_i;
         end
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
         3'b000   : imm_ext_o = {{20{inst_i[31]}}, inst_i[31:20]}; // I-type
         3'b001   : imm_ext_o = {{20{inst_i[31]}}, inst_i[7], inst_i[30:25], inst_i[11:8], 1'b0}; // B-type
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
   output                     alu_zero_o, // Sonuç sıfır sinyali
   output reg [31:0]          alu_out_o   // Sonuç değeri
);

   // Birinci işlenen iki buyruk formatında da sabit.
   wire signed [31:0] alu_a = reg_a_i;
   // İkinci işlenen seçim sinyaline göre belirleniyor.
   wire signed [31:0] alu_b = alu_sel_i ? imm_ext_i : reg_b_i;

   // Sonuç 0'a eşit ise alu_zero_o sinyali 1 olur.
   assign alu_zero_o = ~(|alu_out_o);

   always @(*) begin
      case (alu_fun_i)
         4'b0000  : alu_out_o = alu_a + alu_b;           // Toplama 
         4'b0001  : alu_out_o = alu_a - alu_b;           // Çıkarma
         4'b0010  : alu_out_o = alu_a & alu_b;           // VE
         4'b0011  : alu_out_o = alu_a ^ alu_b;           // XOR
         4'b0100  : alu_out_o = alu_a | alu_b;           // VEYA
         4'b0101  : alu_out_o = alu_a << alu_b[4:0];     // Sola kaydırma
         4'b0110  : alu_out_o = alu_a >> alu_b[4:0];     // Sağa kaydırma
         4'b0111  : alu_out_o = alu_a >>> alu_b[4:0];    // Aritmetik sağa kaydırma
         4'b1000  : alu_out_o = {31'b0, alu_a == alu_b}; // Eşitse alu_out_o = 1, değilse alu_out_o = 0 (beq, bne)
         4'b1001  : alu_out_o = {31'b0, alu_a < alu_b};  // Küçükse alu_out_o = 1, değilse alu_out_o = 0 (blt, bge, slt, slti)
         4'b1010  : alu_out_o = {31'b0, $unsigned(alu_a) < $unsigned(alu_b)}; // (İşaretsiz) küçükse alu_out_o = 1, değilse alu_out_o = 0 (bltu, bgeu, sltu, sltiu)
         default  : alu_out_o = 32'bx;                   // Geçersiz alu_fun_i sinyali
      endcase
   end

endmodule

module address_calculator (
   input [31:0]               pc_i,       // Boru hattı kaydedicisinden gelen program sayacının değeri
   input [31:0]               imm_ext_i,  // Çözme biriminden gelen ivedi değer
   output [31:0]              pc_ext_o    // Program sayacına yazılacak adres
);

   assign pc_ext_o = pc_i + imm_ext_i;

endmodule

module writeback (
   input  [31:0]              alu_out_i,
   output [31:0]              result_o
);

   assign result_o = alu_out_i;

endmodule

module controller (
   input [31:0]               inst_i,        // Boru hattı kaydedicisinden gelen buyruk
   input                      alu_zero_i,    // ALU'dan gelen sonuç sıfır sinyali
   output                     regfile_wen_o, // Kaydedici dosyası yazma yetkilendirme sinyali
   output [2:0]               imm_ext_sel_o, // İvedi genişletici format seçim sinyali
   output                     alu_sel_o,     // ALU ikinci işlenen seçim sinyali
   output reg [3:0]           alu_fun_o,     // ALU işlem seçim sinyali
   output                     pc_sel_o,      // Program sayacı adres seçim sinyali
   output                     clear_o        // Boru hattı boşaltma sinyali
);

   // Buyruğun gerekli bölümleri ayıklanıyor.
   wire [6:0] opcode = inst_i[6:0];
   wire [2:0] funct3 = inst_i[14:12];
   wire [6:0] funct7 = inst_i[31:25];

   wire [1:0] alu_dec;
   wire branch_op;

   reg [7:0] control_signals;
   assign {regfile_wen_o, imm_ext_sel_o, alu_sel_o, alu_dec, branch_op} = control_signals;

   always @(*) begin
      case (opcode)
         7'b0110011  : control_signals = 8'b1_xxx_0_11_0; // R-type buyruk
         7'b0010011  : control_signals = 8'b1_000_1_11_0; // I-type buyruk
         7'b1100011  : control_signals = 8'b0_001_0_01_1; // B-type buyruk
         7'b0000000  : control_signals = 8'b0_000_0_00_0; // Sıfırlama durumu
         default     : control_signals = 8'bx_xxx_x_xx_x; // Geçersiz buyruk
      endcase
   end

   // Buyruk R-type ise ve funct7 değeri 0x20 ise çıkarma işlemi anlamına gelir.
   wire sub = opcode[5] & funct7[5];

   // ALU'da yapılacak işlem belirleniyor.
   always @(*) begin
      case (alu_dec)
         2'b01    : // B-type
            case (funct3)
               3'b000   : alu_fun_o = 4'b1000; // beq
               3'b001   : alu_fun_o = 4'b1000; // bne
               3'b100   : alu_fun_o = 4'b1001; // blt
               3'b101   : alu_fun_o = 4'b1001; // bge
               3'b110   : alu_fun_o = 4'b1010; // bltu
               3'b111   : alu_fun_o = 4'b1010; // bgeu
               default  : alu_fun_o = 4'bx;
            endcase
         2'b11    : // R-type veya I-type
            case (funct3)
               3'b000   : // add-addi veya sub buyruğu
                  if (sub) begin
                     alu_fun_o = 4'b0001; // sub
                  end else begin
                     alu_fun_o = 4'b0000; // add, addi
                  end
               3'b001   : alu_fun_o = 4'b0101; // sll, slli
               3'b010   : alu_fun_o = 4'b1001; // slt, slti
               3'b011   : alu_fun_o = 4'b1010; // sltu, sltiu
               3'b100   : alu_fun_o = 4'b0011; // xor, xori
               3'b101   : // srl, srli, sra, srai
                  if (funct7[5]) begin
                     alu_fun_o = 4'b0111; // sra, srai
                  end else begin
                     alu_fun_o = 4'b0110; // srl, srli
                  end
               3'b110   : alu_fun_o = 4'b0100; // or, ori
               3'b111   : alu_fun_o = 4'b0010; // and, andi
               default  : alu_fun_o = 4'b0000;
            endcase
         default  : alu_fun_o = 4'b0000; // Varsayılan işlem toplama
      endcase
   end

   reg branch_valid;

   always @(*) begin
      case (funct3)
         3'b000   : branch_valid = !alu_zero_i;   // beq
         3'b001   : branch_valid = alu_zero_i;    // bne
         3'b100   : branch_valid = !alu_zero_i;   // blt
         3'b101   : branch_valid = alu_zero_i;    // bge
         3'b110   : branch_valid = !alu_zero_i;   // bltu
         3'b111   : branch_valid = alu_zero_i;    // bgeu
         default  : branch_valid = 1'b0;
      endcase
   end

   assign pc_sel_o   = branch_op & branch_valid; // Dallanma durumu kontrol ediliyor.
   assign clear_o    = pc_sel_o; // Boru hattını boşalt

endmodule
```

</details>

<p align="center">
  <img src="/matrak2.png"/>
</p>

<p align="center">
  Matrak işlemcimizin üst modül şeması
</p>

Modüllerin geri kalanında şimdilik herhangi bir değişiklik yapmayacağız.

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

## Benzetim

Tasarımımız hazır! Hemen basit bir assembly testi yazıp çalışmamızı doğrulayalım.

```
start:
   addi  x5, x0, 4      # x5 = 0x4
   addi  x6, x0, 8      # x6 = 0x8
   beq   x5, x6, main   # x5 ve x6 eşit değil, dallanma gerçekleşmez.
   slli  x5, x5, 1      # x5 = 0x8
   bne   x5, x6, main   # x5 ve x6 eşit, dallanma gerçekleşmez.
   blt   x5, x6, main   # x5, x6'dan küçük değil, dallanma gerçekleşmez.
   beq   x5, x6, main   # x5 ve x6 eşit, dallanma gerçekleşir.
   sll   x5, x5, x5     # Dallanma gerçekleşeceği için bu buyruk işleme alınmaz.

main:
   addi  x7, x0, 47     # x7 = 0x2f
   slt   x4, x5, x7     # x5, x7'den küçük, x4 = 0x1 
```

Test kodumuzun makine koduna dönüştürülmüş hali böyle görünüyor:

program.mem:

```
00400293
00800313
00628c63
00129293
00629863
0062c663
00628463
005292b3
02f00393
0072a233
```

<p align="center">
  <img src="/mvivado2.png"/>
</p>

<p align="center">
  Vivado ile benzetim sonucu
</p>

İşlemcimizi bir önceki bölümde hazırlamış olduğumuz Vivado benzetim ortamında inceliyoruz. Güzel haber! işlemcimiz beklediğimiz şekilde çalıştı ve doğru şekilde dallandı.

> :pushpin: Dalgaformu ekranında durumu bilinmeyen sinyaller, kırmızı x'ler olarak görünüyor. Bu biraz ürkütücü gelse de aslında herşey normal. Başlangıçtaki bilinmeyen sinyaller sıfırlamanın (resetlemenin) ardından olması gereken değerlerini alıyor. Kaydedici dosyasında herhangi bir sıfırlama mekanizması olmaması sebebiyle, değer atanmayan kaydediciler bilinmeyen olarak gözüküyor. Koddaki tüm buyruklar işlendikten sonra bilinmeyen sinyaller tekrar ortaya çıkmaya başlıyor. Bunun sebebi, benzetim ortamında belleğimizin boş kısımlarının bilinmeyen değerlerle dolu olması. Biz bilinmeyen durumların oluşmasını engellemek için ilerleyen bölümlerde yazılım (firmware) kısmında bazı basit önlemler alacağız.

<p align="center">
  <img src="/proper_testbench.jpg"/>
</p>

Bir sonraki bölümde görüşmek dileğiyle\.\.\.

> :scroll: Bu bölümün kodlarına erişmek için [tıklayın](https://github.com/necaticakaci/matrak/tree/main/episodes/ep2).
