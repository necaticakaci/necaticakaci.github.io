---
title: "RISC-V İşlemci Tasarımı - Bölüm 4: Load ve Store Buyrukları"
date: 2023-10-20
description: Bu bölümde load-store buyruklarını ekliyor ve RISC-V işlemcimizi tamamlıyoruz.
math: true
---

Bu bölümde load ve store buyruklarını ekleyip işlemci tasarımımızı tamamlayacağız. load buyruğu bellekten veri okur ve bir hedef kaydedicisine yazar. store buyruğu ise bir kaydedicinin değerini belleğe yazmak için kullanılır. load buyrukları I-type formatında, store buyrukları S-type formatındadır.

**Load Buyrukları:**

| Buyruk          | Tanım                             | opcode          | func3  |
| --------------- | --------------------------------- | --------------- | ------ |
| lb              | rd = mem[rs1 + imm][7:0]          | 0000011         | 0x0    |
| lh              | rd = mem[rs1 + imm][15:0]         | 0000011         | 0x1    |
| lw              | rd = mem[rs1 + imm][31:0]         | 0000011         | 0x2    |
| lbu             | rd = u(mem[rs1 + imm][7:0])       | 0000011         | 0x4    |
| lhu             | rd = u(mem[rs1 + imm][15:0])      | 0000011         | 0x5    |

<code>lb</code>: load byte (bayt yükle)

<code>lh</code>: load half (yarım sözcük yükle)

<code>lw</code>: load word (sözcük yükle)

<code>lbu</code>: load byte unsigned (işaretsiz bayt yükle)

<code>lhu</code>: load half unsigned (işaretsiz yarım sözcük yükle)

> :pushpin: load buyruklarının işaretli ve işaretsiz olarak ayrıldığını görüyoruz. Yükleme işlemi işaretli ve işaretsiz olarak nasıl yapılabilir? Bu noktada işlemcimizin 32-bit değerlerle çalıştığını aklımızdan çıkarmayalım. Bellekten bayt (8-bit) ve yarım sözcük (16-bit) okunduğunda bu değerin 32-bit kaydedicilere yerleştirilmesi için genişletilmesi gerekir. İşaretli sayılar en anlamlı biti ile genişletilirken, işaretsiz sayılar sıfır ile genişletilir.

> :question: Peki neden lwu buyruğu yok? Aslında bu sorunun cevabını yukarıdaki nottan anlayabiliriz. İşlemcimiz 32-bit sayılarla çalıştığı için, lw buyruğu ile bellekten okunan değere herhangi bir genişletme uygulanmaz.

> :pushpin: load buyruklarında okuma yapılacak bellek adresi <code>rs1 + imm</code> işlemiyle hesaplanır. Bu işlemi ALU'da gerçekleştireceğiz.

**Store Buyrukları:**

| Buyruk          | Tanım                             | opcode          | func3  |
| --------------- | --------------------------------- | --------------- | ------ |
| sb              | mem[rs1 + imm][7:0] = rs2[7:0]    | 0100011         | 0x0    |
| sh              | mem[rs1 + imm][15:0] = rs2[15:0]  | 0100011         | 0x1    |
| sw              | mem[rs1 + imm][31:0] = rs2[31:0]  | 0100011         | 0x2    |

<code>sb</code>: store byte (bayt depola)

<code>sh</code>: store half (yarım sözcük depola)

<code>sw</code>: store word (sözcük depola)

> :pushpin: Store buyruklarında genişletme söz konusu değil, dolayısıyla işaretsiz store buyrukları yok.

**I-type Buyruk Formatı:**

| Bitler  | 31 - 20           | 19 - 15 | 14 - 12 | 11 - 7  | 6 - 0   |
| ------- | ----------------- | ------- | ------- | ------- | ------- |
| I-type  | imm[11:0]         | rs1     | funct3  | rd      | opcode  |

**S-type Buyruk Formatı:**

| Bitler  | 31 - 25   | 24 - 20 | 19 - 15 | 14 - 12 | 11 - 7    | 6 - 0   |
| ------- | --------- | ------- | ------- | ------- | --------- | ------- |
| S-type  | imm[11:5] | rs2     | rs1     | funct3  | imm[4:0]  | opcode  |

## Bellek

Bu sefer bellekten başladık. Store buyrukları, 32 bitlik bellek satırımızın 4 baytına da bağımsız olarak erişmemizi gerektiriyor. Bunun için 4-bit <code>stb_i</code> sinyali tanımladık. Bu sayede belleğe kaç bitlik veri yazacağımızı seçebileceğiz.

```verilog
module memory (
   input          clk_i,
   input          wen_i,   // Yazma yetkilendirme girişi
   input [3:0]    stb_i,   // Bayt seçim girişi
   input [31:0]   addr_i,  // Adres girişi
   input [31:0]   data_i,  // Yazılacak veri
   input [31:0]   data_o   // Okunan veri
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
         if (stb_i[0]) mem[addr_i[31:2]][0+:8]  <= data_i[0+:8];
         if (stb_i[1]) mem[addr_i[31:2]][8+:8]  <= data_i[8+:8];
         if (stb_i[2]) mem[addr_i[31:2]][16+:8] <= data_i[16+:8];
         if (stb_i[3]) mem[addr_i[31:2]][24+:8] <= data_i[24+:8];
      end
   end

endmodule
```

> :pushpin: Bellek tasarımını basitleştirmek için ana belleğimizi bir RAM olarak tasarladık. Fakat önbellekleri olmayan bir işlemci için bu durum ideal değil. Çünkü bir portlu RAM belleğimizden çevrim başına bir okuma yapılabiliyoruz. Buyruğun ve verinin aynı çevrimde okunması gerektiği durumlarda (load-store buyruğu işlenirken) işlemciyi durduracağız ve bu durum bize performans kaybı olarak yansıyacak. Belleği iki portlu RAM veya RAM+ROM yaparak işlemciyi durdurmadan kolaylıkla çalıştırabilirdik. Fakat bazen işlemcinin durdurulması gerekebilir ve bunu nasıl yapacağımıza değinmek istiyoruz.

## Getirme Birimi

Getirme birimimizde yapacağımız tek değişiklik program sayacına durdurma mekanizması eklemek olacak. <code>stall_i</code> sinyalinin bir olduğu çevrimde program sayacı güncellenmeyecek.

```verilog
module fetch (
   input                clk_i,
   input                rst_i,
   input                stall_i,    // Program sayacı durdurma girişi
   input                pc_sel_i,   // Program sayacı seçim girişi
   input [31:0]         pc_ext_i,   // Dallanma adres girişi
   output reg [31:0]    pc_o,       // Program sayacı çıkışı
   output [31:0]        pc_plus_o   // Program sayacı + 4 çıkışı
);

   // Program sayacına 4 ekle
   assign pc_plus_o = pc_o + 4;

   // Dallanma adresi veya PC + 4 
   wire [31:0] pc_next = pc_sel_i ? pc_ext_i : pc_plus_o;

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         pc_o <= 32'h0000_0000;
      end else begin
         if (!stall_i) begin // Durdurma isteği yoksa PC'yi güncelle
            pc_o <= pc_next;
         end
      end
   end

endmodule
```

## Boru Hattı Kaydedicileri

Boru hattı kaydedicilerinde değişiklik yapmamızı gerektirecek bir durum yok. Önceki bölümde tasarladığımız modülü kullanmaya devam edeceğiz.

```verilog
module fd_regs (
   input                clk_i,
   input                rst_i,
   input                clear_i,     // Sıfırlama sinyali (boru hattı boşaltma)
   input [31:0]         inst_f_i,    // Buyruk girişi (bellekten geliyor)
   input [31:0]         pc_f_i,      // Progam sayacı girişi (getirme biriminden geliyor)
   input [31:0]         pc_plus_f_i, // Program sayacı + 4 girişi (getirme biriminden geliyor)
   output reg [31:0]    inst_d_o,    // Buyruk çıkışı (yürütme aşamasına gidiyor)
   output reg [31:0]    pc_d_o,      // Program sayacı çıkışı (yürütme aşamasına gidiyor)
   output reg [31:0]    pc_plus_d_o  // Program sayacı + 4 çıkışı (yürütme aşamasına gidiyor)
);

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         inst_d_o    <= 32'b0;
         pc_d_o      <= 32'b0;
         pc_plus_d_o <= 32'b0;
      end else begin
         if (clear_i) begin // Boru hattı boşaltılıyor.
            inst_d_o    <= 32'b0;
            pc_d_o      <= 32'b0;
            pc_plus_d_o <= 32'b0;
         end else begin
            inst_d_o    <= inst_f_i;
            pc_d_o      <= pc_f_i;
            pc_plus_d_o <= pc_plus_f_i;
         end
      end
   end

endmodule
```

## Çözme Birimi

Son buyruk formatımız S-type'ı da ivedi genişleticiye ekliyoruz.

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
         3'b010   : imm_ext_o = {{12{inst_i[31]}}, inst_i[19:12], inst_i[20], inst_i[30:21], 1'b0}; // J-type
         3'b011   : imm_ext_o = {inst_i[31:12], 12'b0}; // U-type
         3'b100   : imm_ext_o = {{20{inst_i[31]}}, inst_i[31:25], inst_i[11:7]}; // S-type
         default  : imm_ext_o = 32'b0; 
      endcase
   end

endmodule
```

## ALU

ALU tasarımımız çoktan bitmişti. Yine de tekrar bir göz atalım.

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

Adres hesaplayıcı da son halini aldı. Hiç bir şey değişmeyecek. ¯\\\_(ツ)_/¯

```verilog
module address_calculator (
   input                      ac_sel_i,   // Kontrol biriminden gelen kaynak seçim sinyali
   input [31:0]               pc_i,       // Boru hattı kaydedicisinden gelen program sayacının değeri
   input [31:0]               imm_ext_i,  // Çözme biriminden gelen ivedi değer
   input [31:0]               reg_a_i,    // Çözme biriminden gelen rs1 değeri
   output [31:0]              pc_ext_o    // Program sayacına yazılacak adres
);

   wire [31:0] operand = ac_sel_i ? reg_a_i : pc_i;

   assign pc_ext_o = operand + imm_ext_i;

endmodule
```

## Geriyazma Birimi

Load buyruklarında bellekten okunan değerin bir hedef kaydedicisine yazılması gerekiyor. Kaydediciye yazılacak kaynağı geriyazma birimi ile seçtiğimiz için burada biraz düzenleme yapmalıyız. Bellekten okuma işlemini birazdan yazacağımız yükleme depolama birimi gerçekleştirecek. Geriyazma birimi, bellekten okunan değeri <code>ls_rdata_i</code> sinyali aracılığıyla kaydedici dosyasına yazabilecek. Ayrıca <code>result_sel_i</code> seçim sinyalimiz artık 3-bit genişlikte olmalı.

```verilog
module writeback (
   input [2:0]                result_sel_i,  // Kontrol biriminden gelen seçim sinyali
   input [31:0]               alu_out_i,     // ALU sonucu
   input [31:0]               pc_plus_i,     // Program sayacı + 4
   input [31:0]               imm_ext_i,     // İvedi değer
   input [31:0]               pc_ext_i,      // Adres hesaplayıcıdan gelen adres
   input [31:0]               ls_rdata_i,    // Bellekten okunan değer
   output reg [31:0]          result_o       // Kaydedici dosyasına yazılacak değer
);

   always @(*) begin
      case (result_sel_i)
         3'b000   : result_o = alu_out_i;
         3'b001   : result_o = pc_plus_i;
         3'b010   : result_o = imm_ext_i;
         3'b011   : result_o = pc_ext_i;
         3'b100   : result_o = ls_rdata_i;
         default  : result_o = 32'bx;
      endcase
   end

endmodule
```

## Yükleme Depolama Birimi

Load ve Store buyrukları için yeni bir modül ekliyoruz. Bu birimin görevi bellekte okuma ve yazma işlemleri gerçekleştirmek olacak.

> :pushpin: Hatırlayacağınız üzere bellek satırlarımızı 32-bit genişlikte olacak şekilde tasarlamıştık. Belleğe 32-bit genişlikte bir değer yazacağımızda herhangi bir sorunla karşılaşmayacağız çünkü satırın tamamını kaplayacak. Ancak 16-bit genişlikte bir değer (yarım sözcük) satıra iki farklı (alt veya üst 16 bit) şekilde yerleşebilir. Benzer durum olarak bir bayt 4 farklı şekilde yerleşebilir. Yükleme depolama birimimizde, bu hizalamayı adrese bakarak sağlayacağız.

> :pushpin: Bayt hizalı erişimden bahsederken, konuyu hizasız erişim (unaligned access) ile karıştırmayalım. Hizasız erişim, verinin birden fazla adres satırına yerleştirilmesi durumunda ortaya çıkar. Örneğin, 32-bit sayının dördün katı olmayan 0x5 adresine yerleştirildiğinde, bu sayıya erişmek için iki satırın okunması ve değerlerin doğru şekilde birleştirilmesi gerekir. Tasarladığımız işlemci şimdilik hizasız erişimleri desteklemeyecek.

32-bit genişlikte 3 satırı olan bir belleğimiz olduğunu farzedelim. Başlangıç değerleri aşağıdaki gibi olsun.

| Adres  | Değer (32-bit)  |
| ------ | --------------- |
| 0x0    | 0x<abbr title="Adres: 0x3">ff_</abbr><abbr title="Adres: 0x2">ff_</abbr><abbr title="Adres: 0x1">ff_</abbr><abbr title="Adres: 0x0">ff</abbr> |
| 0x4    | 0x<abbr title="Adres: 0x7">ff_</abbr><abbr title="Adres: 0x6">ff_</abbr><abbr title="Adres: 0x5">ff_</abbr><abbr title="Adres: 0x4">ff</abbr> |
| 0x8    | 0x<abbr title="Adres: 0xb">ff_</abbr><abbr title="Adres: 0xa">ff_</abbr><abbr title="Adres: 0x9">ff_</abbr><abbr title="Adres: 0x8">ff</abbr> |

Aşağıdaki assembly kodunun işlenmesinin ardından belleğin durumunu inceleyelim. Buyrukların farklı bellek uzayından geldiğini kabul ediyoruz.

```
li x8, 0x25       # x8 = 0x25
li x6, 0xfeda     # x6 = 0xfeda
li x4, 0xcefa     # x4 = 0xcefa
li x2, 0x02312313 # x2 = 0x02312313

sb x8, 5(x0)      # mem[5] = 0x25
sh x6, 6(x0)      # mem[6] = 0xfeda
sh x4, 3(x0)      # mem[3] = 0xcefa, yarım sözcük hizasız yerleştirildi!
sw x2, 8(x0)      # mem[8] = 0x02312313
```

Buyruklar işlendikten sonra belleğimiz aşağıdaki gibi gözükür.

| Adres  | Değer (32-bit)  |
| ------ | --------------- |
| 0x0    | 0x<abbr title="Adres: 0x3">fa_</abbr><abbr title="Adres: 0x2">ff_</abbr><abbr title="Adres: 0x1">ff_</abbr><abbr title="Adres: 0x0">ff</abbr> |
| 0x4    | 0x<abbr title="Adres: 0x7">fe_</abbr><abbr title="Adres: 0x6">da_</abbr><abbr title="Adres: 0x5">25_</abbr><abbr title="Adres: 0x4">ce</abbr> |
| 0x8    | 0x<abbr title="Adres: 0xb">02_</abbr><abbr title="Adres: 0xa">31_</abbr><abbr title="Adres: 0x9">23_</abbr><abbr title="Adres: 0x8">13</abbr> |

> :bulb: Bellek hizalama tek seferde anlaşılması zor bir konu. İnternetten çalışan [Venus](https://venus.cs61c.org/) ve [Ripes](https://ripes.me/) gibi öykünücülerle biraz antrenman yapmak iyi fikir olabilir.

Şimdi yükleme depolama birimimize geçebiliriz.

```verilog
module load_store (
   input                      ls_wen_i,   // Yazma yetkilendirme, kontrol biriminden geliyor.
   input                      ls_ren_i,   // Okuma yetkilendirme, kontrol biriminden geliyor.
   input [2:0]                ls_fmt_i,   // funct3 değeri, kontrol biriminden geliyor.
   input [31:0]               ls_addr_i,  // Adres girişi, ALU'dan geliyor.
   input [31:0]               ls_wdata_i, // rs2 değeri, çözme biriminden geliyor.
   input [31:0]               ls_rdata_i, // Bellekten okunan değer.
   output                     ls_wen_o,   // Yazma yetkilendirme, belleğe gidiyor.
   output                     ls_ren_o,   // Okuma yetkilendirme, belleğe gidiyor.
   output reg [3:0]           ls_stb_o,   // Bayt seçim sinyali, belleğe gidiyor.
   output [31:0]              ls_addr_o,  // Adres çıkışı, belleğe gidiyor.
   output [31:0]              ls_wdata_o, // Yazılacak veri, belleğe gidiyor.
   output reg [31:0]          ls_rdata_o  // Okunan veri, geriyazma birimine gidiyor.
);

   // Bu sinyaller doğrudan belleğe gidiyor.
   assign ls_addr_o  = ls_addr_i;
   assign ls_wen_o   = ls_wen_i;
   assign ls_ren_o   = ls_ren_i;

   // Kaydırılacak değer hesaplanıyor.
   wire [4:0] shift_value = ls_addr_i[1:0] << 5'd3;

   // Belleğe yazılacak veri hizalanıyor.
   assign ls_wdata_o = ls_wdata_i << shift_value;

   // Bellekten okunan veri hizalanıyor.
   wire [31:0] aligned_data = ls_rdata_i >> shift_value;

   // Bellekten okunan hizalanmış veri genişletiliyor.
   always @(*) begin
      case (ls_fmt_i[1:0])
         2'b00    : ls_rdata_o = {{24{~ls_fmt_i[2] & aligned_data[7]}}, aligned_data[7:0]};     // lb, lbu
         2'b01    : ls_rdata_o = {{16{~ls_fmt_i[2] & aligned_data[15]}}, aligned_data[15:0]};   // lh, lhu
         2'b10    : ls_rdata_o = aligned_data[31:0];  // lw
         default  : ls_rdata_o = 32'bx;
      endcase
   end

   // Bayt seçim sinyali ayarlanıyor.
   always @(*) begin
      case (ls_fmt_i[1:0])
         2'b00    : ls_stb_o = 4'b0001 << ls_addr_i[1:0];   // sb
         2'b01    : ls_stb_o = 4'b0011 << ls_addr_i[1:0];   // sh
         2'b10    : ls_stb_o = 4'b1111 << ls_addr_i[1:0];   // sw
         default  : ls_stb_o = 4'b0;
      endcase
   end

endmodule
```

## Kontrol Birimi

Kontrol birimimizde load ve store buyruklarını desteklemek için opcode çözücümüzü güncelliyoruz. Yeni kontrol sinyallerimiz:

<code>ls_wen_o</code>: Bellek yazma yetkilendirme etkin.

<code>ls_ren_o</code>: Bellek okuma yetkilendirme etkin.

<code>ls_fmt_o</code>: Yükleme depolama birimi için funct3 değeri.

```verilog
module controller (
   input [31:0]               inst_i,        // Boru hattı kaydedicisinden gelen buyruk
   input                      alu_zero_i,    // ALU'dan gelen sonuç sıfır sinyali
   output                     regfile_wen_o, // Kaydedici dosyası yazma yetkilendirme sinyali
   output [2:0]               imm_ext_sel_o, // İvedi genişletici format seçim sinyali
   output                     alu_sel_o,     // ALU ikinci işlenen seçim sinyali
   output reg [3:0]           alu_fun_o,     // ALU işlem seçim sinyali
   output                     pc_sel_o,      // Program sayacı adres seçim sinyali
   output                     ac_sel_o,      // Adres hesaplayıcı kaynak seçim sinyali
   output [2:0]               result_sel_o,  // Geriyazma kaynak seçim sinyali
   output                     ls_wen_o,      // Bellek yazma yetkilendirme sinyali
   output                     ls_ren_o,      // Bellek okuma yetkilendirme sinyali
   output [2:0]               ls_fmt_o,      // Yükleme depolama birimi için funct3 sinyali
   output                     clear_o        // Boru hattı boşaltma sinyali
);

   // Buyruğun gerekli bölümleri ayıklanıyor.
   wire [6:0] opcode = inst_i[6:0];
   wire [2:0] funct3 = inst_i[14:12];
   wire [6:0] funct7 = inst_i[31:25];

   assign ls_fmt_o = funct3;

   wire [1:0] alu_dec;
   wire branch_op;
   wire jump_op;

   reg [14:0] control_signals;
   assign {regfile_wen_o, imm_ext_sel_o, alu_sel_o, alu_dec, branch_op, jump_op, ac_sel_o, result_sel_o, ls_wen_o, ls_ren_o} = control_signals;

   always @(*) begin
      case (opcode)
         7'b0110011  : control_signals = 15'b1_xxx_0_11_0_0_0_000_0_0; // R-type buyruk
         7'b0010011  : control_signals = 15'b1_000_1_11_0_0_0_000_0_0; // I-type buyruk
         7'b1100011  : control_signals = 15'b0_001_0_01_1_0_0_000_0_0; // B-type buyruk
         7'b1101111  : control_signals = 15'b1_010_0_00_0_1_0_001_0_0; // jal
         7'b1100111  : control_signals = 15'b1_000_0_00_0_1_1_001_0_0; // jalr
         7'b0110111  : control_signals = 15'b1_011_0_00_0_0_0_010_0_0; // lui
         7'b0010111  : control_signals = 15'b1_011_0_00_0_0_0_011_0_0; // auipc
         7'b0000011  : control_signals = 15'b1_000_1_10_0_0_0_100_0_1; // load buyrukları
         7'b0100011  : control_signals = 15'b0_100_1_10_0_0_0_000_1_0; // store buyrukları
         7'b0000000  : control_signals = 15'b0_000_0_00_0_0_0_000_0_0; // Sıfırlama durumu
         default     : control_signals = 15'bx_xxx_x_xx_x_x_x_xxx_x_x; // Geçersiz buyruk
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

   assign pc_sel_o   = (branch_op & branch_valid) | jump_op; // Dallanma ve atlama durumu kontrol ediliyor.
   assign clear_o    = pc_sel_o; // Boru hattını boşalt

endmodule
```

## Parçaları Bir Araya Getirme

Nihayet RV32I çekirdeğimizin tüm modüllerine son şeklini vermiş olduk. Şimdi çekirdek modüllerini bir araya getirmeliyiz.

Yeni oluşan bellek arayüz sinyallerini ve işlemci durdurma sinyalini matrak modülünün dışına çıkartıyoruz. İşlemci durdurulduğunda, ikinci aşamada herhangi bir yürütme işlemi gerçekleştirilmemesi için boru hattının temizlenmesi gerekiyor. Kontrol biriminden gelen temizleme sinyali ile çekirdek dışından gelen durdurma sinyalini VEYA işlemine tabi tutarak, boru hattı kaydedicisinin temizleme girişine bağlıyoruz.

<details>
<summary>matrak.v: <mark>kodu göstermek için tıklayın</mark></summary>

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Processor Module

module matrak (
   input                clk_i,
   input                rst_i,
   input                stall_i,       // İşlemci durdurma sinyali
   input [31:0]         inst_i,        // Buyruk girişi
   input [31:0]         data_i,        // Veri girişi
   output               wen_o,         // Yazma yetkilendirme
   output               ren_o,         // Okuma yetkilendirme
   output [3:0]         stb_o,         // Bayt seçim sinyali
   output [31:0]        inst_addr_o,   // Buyruk adresi
   output [31:0]        data_addr_o,   // Veri adresi
   output [31:0]        data_o         // Belleğe yazılacak veri
);

   // Getirme birimi bağlantıları
   wire c2f_pc_sel;
   wire [31:0] ac2f_pc_ext;
   wire [31:0] f2fd_pc_plus;

   fetch f1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .stall_i(stall_i),
      .pc_sel_i(c2f_pc_sel),
      .pc_ext_i(ac2f_pc_ext),
      .pc_o(inst_addr_o),
      .pc_plus_o(f2fd_pc_plus)
   );

   // Boru hattı kaydedicisi bağlantıları
   wire [31:0] fd2d_inst;
   wire [31:0] fd2ac_pc;
   wire [31:0] fd2w_pc_plus;
   wire c2fd_clear;

   // Boru hattı temizleme sinyali
   wire clear = c2fd_clear | stall_i;

   fd_regs fd1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .clear_i(clear),
      .inst_f_i(inst_i),
      .pc_f_i(inst_addr_o),
      .pc_plus_f_i(f2fd_pc_plus),
      .inst_d_o(fd2d_inst),
      .pc_d_o(fd2ac_pc),
      .pc_plus_d_o(fd2w_pc_plus)
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

   // Adres hesaplayıcı bağlantıları
   wire c2ac_ac_sel;

   address_calculator ac1(
      .ac_sel_i(c2ac_ac_sel),
      .pc_i(fd2ac_pc),
      .imm_ext_i(d2a_imm_ext),
      .reg_a_i(d2a_reg_a),
      .pc_ext_o(ac2f_pc_ext)
   );

   // Geriyazma birimi bağlantıları
   wire [2:0] c2w_result_sel;
   wire [31:0] ls2w_rdata;

   writeback w1 (
      .result_sel_i(c2w_result_sel),
      .alu_out_i(a2w_alu_out),
      .pc_plus_i(fd2w_pc_plus),
      .imm_ext_i(d2a_imm_ext),
      .pc_ext_i(ac2f_pc_ext),
      .ls_rdata_i(ls2w_rdata),
      .result_o(w2d_result)
   );

   // Yükleme depolama birimi bağlantıları
   wire c2ls_wen;
   wire c2ls_ren;
   wire [2:0] c2ls_fmt;

   load_store ls1 (
      .ls_wen_i(c2ls_wen),
      .ls_ren_i(c2ls_ren),
      .ls_fmt_i(c2ls_fmt),
      .ls_addr_i(a2w_alu_out),
      .ls_wdata_i(d2a_reg_b),
      .ls_rdata_i(data_i),
      .ls_wen_o(wen_o),
      .ls_ren_o(ren_o),
      .ls_stb_o(stb_o),
      .ls_addr_o(data_addr_o),
      .ls_wdata_o(data_o),
      .ls_rdata_o(ls2w_rdata)
   );

   controller c1 (
      .inst_i(fd2d_inst),
      .alu_zero_i(a2c_alu_zero),
      .regfile_wen_o(c2d_regfile_wen),
      .imm_ext_sel_o(c2d_imm_ext_sel),
      .alu_sel_o(c2a_alu_sel),
      .alu_fun_o(c2a_alu_fun),
      .pc_sel_o(c2f_pc_sel),
      .ac_sel_o(c2ac_ac_sel),
      .result_sel_o(c2w_result_sel),
      .ls_wen_o(c2ls_wen),
      .ls_ren_o(c2ls_ren),
      .ls_fmt_o(c2ls_fmt),
      .clear_o(c2fd_clear)
   );

endmodule

module fetch (
   input                clk_i,
   input                rst_i,
   input                stall_i,    // Program sayacı durdurma girişi
   input                pc_sel_i,   // Program sayacı seçim girişi
   input [31:0]         pc_ext_i,   // Dallanma adres girişi
   output reg [31:0]    pc_o,       // Program sayacı çıkışı
   output [31:0]        pc_plus_o   // Program sayacı + 4 çıkışı
);

   // Program sayacına 4 ekle
   assign pc_plus_o = pc_o + 4;

   // Dallanma adresi veya PC + 4 
   wire [31:0] pc_next = pc_sel_i ? pc_ext_i : pc_plus_o;

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         pc_o <= 32'h0000_0000;
      end else begin
         if (!stall_i) begin // Durdurma isteği yoksa PC'yi güncelle
            pc_o <= pc_next;
         end
      end
   end

endmodule

module fd_regs (
   input                clk_i,
   input                rst_i,
   input                clear_i,     // Sıfırlama sinyali (boru hattı boşaltma)
   input [31:0]         inst_f_i,    // Buyruk girişi (bellekten geliyor)
   input [31:0]         pc_f_i,      // Progam sayacı girişi (getirme biriminden geliyor)
   input [31:0]         pc_plus_f_i, // Program sayacı + 4 girişi (getirme biriminden geliyor)
   output reg [31:0]    inst_d_o,    // Buyruk çıkışı (yürütme aşamasına gidiyor)
   output reg [31:0]    pc_d_o,      // Program sayacı çıkışı (yürütme aşamasına gidiyor)
   output reg [31:0]    pc_plus_d_o  // Program sayacı + 4 çıkışı (yürütme aşamasına gidiyor)
);

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         inst_d_o    <= 32'b0;
         pc_d_o      <= 32'b0;
         pc_plus_d_o <= 32'b0;
      end else begin
         if (clear_i) begin // Boru hattı boşaltılıyor.
            inst_d_o    <= 32'b0;
            pc_d_o      <= 32'b0;
            pc_plus_d_o <= 32'b0;
         end else begin
            inst_d_o    <= inst_f_i;
            pc_d_o      <= pc_f_i;
            pc_plus_d_o <= pc_plus_f_i;
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
         3'b010   : imm_ext_o = {{12{inst_i[31]}}, inst_i[19:12], inst_i[20], inst_i[30:21], 1'b0}; // J-type
         3'b011   : imm_ext_o = {inst_i[31:12], 12'b0}; // U-type
         3'b100   : imm_ext_o = {{20{inst_i[31]}}, inst_i[31:25], inst_i[11:7]}; // S-type
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
   input                      ac_sel_i,   // Kontrol biriminden gelen kaynak seçim sinyali
   input [31:0]               pc_i,       // Boru hattı kaydedicisinden gelen program sayacının değeri
   input [31:0]               imm_ext_i,  // Çözme biriminden gelen ivedi değer
   input [31:0]               reg_a_i,    // Çözme biriminden gelen rs1 değeri
   output [31:0]              pc_ext_o    // Program sayacına yazılacak adres
);

   wire [31:0] operand = ac_sel_i ? reg_a_i : pc_i;

   assign pc_ext_o = operand + imm_ext_i;

endmodule

module writeback (
   input [2:0]                result_sel_i,  // Kontrol biriminden gelen seçim sinyali
   input [31:0]               alu_out_i,     // ALU sonucu
   input [31:0]               pc_plus_i,     // Program sayacı + 4
   input [31:0]               imm_ext_i,     // İvedi değer
   input [31:0]               pc_ext_i,      // Adres hesaplayıcıdan gelen adres
   input [31:0]               ls_rdata_i,    // Bellekten okunan değer
   output reg [31:0]          result_o       // Kaydedici dosyasına yazılacak değer
);

   always @(*) begin
      case (result_sel_i)
         3'b000   : result_o = alu_out_i;
         3'b001   : result_o = pc_plus_i;
         3'b010   : result_o = imm_ext_i;
         3'b011   : result_o = pc_ext_i;
         3'b100   : result_o = ls_rdata_i;
         default  : result_o = 32'bx;
      endcase
   end

endmodule

module load_store (
   input                      ls_wen_i,   // Yazma yetkilendirme, kontrol biriminden geliyor.
   input                      ls_ren_i,   // Okuma yetkilendirme, kontrol biriminden geliyor.
   input [2:0]                ls_fmt_i,   // funct3 değeri, kontrol biriminden geliyor.
   input [31:0]               ls_addr_i,  // Adres girişi, ALU'dan geliyor.
   input [31:0]               ls_wdata_i, // rs2 değeri, çözme biriminden geliyor.
   input [31:0]               ls_rdata_i, // Bellekten okunan değer.
   output                     ls_wen_o,   // Yazma yetkilendirme, belleğe gidiyor.
   output                     ls_ren_o,   // Okuma yetkilendirme, belleğe gidiyor.
   output reg [3:0]           ls_stb_o,   // Bayt seçim sinyali, belleğe gidiyor.
   output [31:0]              ls_addr_o,  // Adres çıkışı, belleğe gidiyor.
   output [31:0]              ls_wdata_o, // Yazılacak veri, belleğe gidiyor.
   output reg [31:0]          ls_rdata_o  // Okunan veri, geriyazma birimine gidiyor.
);

   // Bu sinyaller doğrudan belleğe gidiyor.
   assign ls_addr_o  = ls_addr_i;
   assign ls_wen_o   = ls_wen_i;
   assign ls_ren_o   = ls_ren_i;

   // Kaydırılacak değer hesaplanıyor.
   wire [4:0] shift_value = ls_addr_i[1:0] << 5'd3;

   // Belleğe yazılacak veri hizalanıyor.
   assign ls_wdata_o = ls_wdata_i << shift_value;

   // Bellekten okunan veri hizalanıyor.
   wire [31:0] aligned_data = ls_rdata_i >> shift_value;

   // Bellekten okunan hizalanmış veri genişletiliyor.
   always @(*) begin
      case (ls_fmt_i[1:0])
         2'b00    : ls_rdata_o = {{24{~ls_fmt_i[2] & aligned_data[7]}}, aligned_data[7:0]};     // lb, lbu
         2'b01    : ls_rdata_o = {{16{~ls_fmt_i[2] & aligned_data[15]}}, aligned_data[15:0]};   // lh, lhu
         2'b10    : ls_rdata_o = aligned_data[31:0];  // lw
         default  : ls_rdata_o = 32'bx;
      endcase
   end

   // Bayt seçim sinyali ayarlanıyor.
   always @(*) begin
      case (ls_fmt_i[1:0])
         2'b00    : ls_stb_o = 4'b0001 << ls_addr_i[1:0];   // sb
         2'b01    : ls_stb_o = 4'b0011 << ls_addr_i[1:0];   // sh
         2'b10    : ls_stb_o = 4'b1111 << ls_addr_i[1:0];   // sw
         default  : ls_stb_o = 4'b0;
      endcase
   end

endmodule

module controller (
   input [31:0]               inst_i,        // Boru hattı kaydedicisinden gelen buyruk
   input                      alu_zero_i,    // ALU'dan gelen sonuç sıfır sinyali
   output                     regfile_wen_o, // Kaydedici dosyası yazma yetkilendirme sinyali
   output [2:0]               imm_ext_sel_o, // İvedi genişletici format seçim sinyali
   output                     alu_sel_o,     // ALU ikinci işlenen seçim sinyali
   output reg [3:0]           alu_fun_o,     // ALU işlem seçim sinyali
   output                     pc_sel_o,      // Program sayacı adres seçim sinyali
   output                     ac_sel_o,      // Adres hesaplayıcı kaynak seçim sinyali
   output [2:0]               result_sel_o,  // Geriyazma kaynak seçim sinyali
   output                     ls_wen_o,      // Bellek yazma yetkilendirme sinyali
   output                     ls_ren_o,      // Bellek okuma yetkilendirme sinyali
   output [2:0]               ls_fmt_o,      // Yükleme depolama birimi için funct3 sinyali
   output                     clear_o        // Boru hattı boşaltma sinyali
);

   // Buyruğun gerekli bölümleri ayıklanıyor.
   wire [6:0] opcode = inst_i[6:0];
   wire [2:0] funct3 = inst_i[14:12];
   wire [6:0] funct7 = inst_i[31:25];

   assign ls_fmt_o = funct3;

   wire [1:0] alu_dec;
   wire branch_op;
   wire jump_op;

   reg [14:0] control_signals;
   assign {regfile_wen_o, imm_ext_sel_o, alu_sel_o, alu_dec, branch_op, jump_op, ac_sel_o, result_sel_o, ls_wen_o, ls_ren_o} = control_signals;

   always @(*) begin
      case (opcode)
         7'b0110011  : control_signals = 15'b1_xxx_0_11_0_0_0_000_0_0; // R-type buyruk
         7'b0010011  : control_signals = 15'b1_000_1_11_0_0_0_000_0_0; // I-type buyruk
         7'b1100011  : control_signals = 15'b0_001_0_01_1_0_0_000_0_0; // B-type buyruk
         7'b1101111  : control_signals = 15'b1_010_0_00_0_1_0_001_0_0; // jal
         7'b1100111  : control_signals = 15'b1_000_0_00_0_1_1_001_0_0; // jalr
         7'b0110111  : control_signals = 15'b1_011_0_00_0_0_0_010_0_0; // lui
         7'b0010111  : control_signals = 15'b1_011_0_00_0_0_0_011_0_0; // auipc
         7'b0000011  : control_signals = 15'b1_000_1_10_0_0_0_100_0_1; // load buyrukları
         7'b0100011  : control_signals = 15'b0_100_1_10_0_0_0_000_1_0; // store buyrukları
         7'b0000000  : control_signals = 15'b0_000_0_00_0_0_0_000_0_0; // Sıfırlama durumu
         default     : control_signals = 15'bx_xxx_x_xx_x_x_x_xxx_x_x; // Geçersiz buyruk
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

   assign pc_sel_o   = (branch_op & branch_valid) | jump_op; // Dallanma ve atlama durumu kontrol ediliyor.
   assign clear_o    = pc_sel_o; // Boru hattını boşalt

endmodule
```

</details>

<p align="center">
  <img src="/matrak4.png"/>
</p>

<p align="center">
  Matrak işlemcimizin üst modül şeması
</p>

Yukarıda tasarladığımız bellek modülümüzü tekrar hatırlayalım.

memory.v:

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Main Memory Module

module memory (
   input          clk_i,
   input          wen_i,   // Yazma yetkilendirme girişi
   input [3:0]    stb_i,   // Bayt seçim girişi
   input [31:0]   addr_i,  // Adres girişi
   input [31:0]   data_i,  // Yazılacak veri
   input [31:0]   data_o   // Okunan veri
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
         if (stb_i[0]) mem[addr_i[31:2]][0+:8]  <= data_i[0+:8];
         if (stb_i[1]) mem[addr_i[31:2]][8+:8]  <= data_i[8+:8];
         if (stb_i[2]) mem[addr_i[31:2]][16+:8] <= data_i[16+:8];
         if (stb_i[3]) mem[addr_i[31:2]][24+:8] <= data_i[24+:8];
      end
   end

endmodule
```

İşlemci ve bellek arasındaki bağlantılarda değişiklikler meydana geldiği için üst modülde de düzenlemeler gerçekleştiriyoruz. İşlemciden okuma veya yazma isteği gelirse, buyruk adresi yerine veri adresini belleğe iletiyoruz ve ardından işlemciyi bir çevrim boyunca durduruyoruz.

> :pushpin: Bir load-store buyruğu yürütme aşamasına ulaştığında, bellekten yeni buyruk okunamayacağı için progam sayacı bir çevrim durdurulur (bir sonraki çevrimde değeri güncellenmez) ve bir sonraki çevrimde boru hattı boşaltılarak nop buyruğu işletilir. Bir çevrim daha sonra ise işlemci normal çalışmasına devam eder.

Tasarladığımız üst modül kodu aşağıda verilmiştir.

top.v:

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Top Module

module top (
   input             clk_i,
   input             rst_i
);

   // İşlemci bağlantıları
   wire stall;
   wire wen;
   wire ren;
   wire [3:0] stb;
   wire [31:0] inst_addr;
   wire [31:0] data_addr;
   wire [31:0] wdata;

   matrak mt1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .stall_i(stall),
      .inst_i(mem_rdata),
      .data_i(mem_rdata),
      .wen_o(wen),
      .ren_o(ren),
      .stb_o(stb),
      .inst_addr_o(inst_addr),
      .data_addr_o(data_addr),
      .data_o(wdata)
   );

   // Bellek bağlantıları
   wire [31:0] mem_addr;
   wire [31:0] mem_rdata;

   memory me1 (
      .clk_i(clk_i),
      .wen_i(wen),
      .stb_i(stb),
      .addr_i(mem_addr),
      .data_i(wdata),
      .data_o(mem_rdata)
   );

   // Load Store isteği kontrol ediliyor.
   wire loadstore_request = wen | ren;

   // Belleğe aktarılacak adres seçiliyor.
   assign mem_addr   = loadstore_request ? data_addr : inst_addr;

   // Bellek meşgulse işlemciyi durdur.
   assign stall      = loadstore_request;

endmodule
```

Benzetim modülümüzde herhangi bir değişiklik yapmayacağız.

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

Load-Store buyruklarını test etmek için assembly ile bir program hazırlıyoruz.

```
start:
   addi x8, x0, 0xb4    # x8 = 0xb4
   addi x2, x0, 0x44    # x2 = 0x44
   sw x8, 0(x2)         # ram[0x44] = 0x000000b4
   lw x9, 0(x2)         # x9 = 0xb4
   add x8, x8, x8       # x8 = 0x168
   addi x2, x2, 0x4     # x2 = 0x48
   sw x8, 0(x2)         # ram[0x48] = 0x00000168
   lw x10, 0(x2)        # x10 = 0x168
   addi x20, x0, 0x7ff  # x20 = 0x7ff
   sb x20, 4(x2)        # ram[0x4c] = 0xff
   sh x20, 8(x2)        # ram[0x50] = 0x07ff
   lb x16, 4(x2)        # x16 = 0xffffffff
   lh x12, 8(x2)        # x12 = 0x7ff
   lbu x13, 4(x2)       # x13 = 0xff
   lhu x14, 8(x2)       # x14 = 0x7ff
```

Hazırladığımız test kodu makine koduna çevrildiğinde aşağıdaki gibi gözüküyor.

program.mem:

```
0b400413
04400113
00812023
00012483
00840433
00410113
00812023
00012503
7ff00a13
01410223
01411423
00410803
00811603
00414683
00815703
```

<p align="center">
  <img src="/mvivado4.png"/>
</p>

<p align="center">
  Vivado ile benzetim sonucu
</p>

Test kodunun işlemcimizde beklediğimiz şekilde çalıştığını gözlemliyoruz.

:tada: Harika! RV32I çekirdeğimizi tamamladık. Peki sırada ne var?

> :scroll: Bu bölümün kodlarına erişmek için [tıklayın](https://github.com/necaticakaci/matrak/tree/main/episodes/ep4).
