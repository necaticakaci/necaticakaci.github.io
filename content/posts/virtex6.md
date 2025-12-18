---
title: "Garip Bir Virtex 6 FPGA Kartı"
date: 2025-12-07
description: Bu yazıda internet üzerinde fazla bilgi bulunmayan bir FPGA kartını inceleyip kullanmaya çalışıyoruz.
math: true
---

Bundan birkaç yıl önce Aliexpress'te gezerken bir FPGA kartı dikkatimi çekti. Listelenen özelliklerine göre fiyatı oldukça uygun görünüyordu. Bununla birlikte kartın ne bir model numarası vardı, ne de internet üzerinde kart hakkında bilgiye rastlamak mümkündü. Neyse ki satıcı, ürünü satın alanlara dokümantasyon ve örnek proje paketi sağlayacağını yazmıştı. Kartın neden bu kadar ucuz olduğunu sorgulamak yerine satın alıp denemenin değeceğini düşündüm. Kart ve dokümantasyon elime geçti fakat uzun süre test etmeye fırsat bulamadım.

Geçen bu sürede gümrük kanunları değişti ve FPGA fiyatları yükseldi. Şimdilerde Aliexpress üzerinde rastladığım Virtex ve Kintex serilerinden kartların Türkiye'ye gönderim seçeneğinin kapalı olduğunu görüyorum ki bu durum üzücü. Sanırım artık bu seviyede bir kartı bireysel olarak yurt dışından sipariş vermek mümkün değil. Ama belki de daha derin bir araştırmayla benzer kartlar bulunabilir.

## Kartın İncelemesi

İlk bakışta kartın PCI Express bağlantısı ve fiber optik iletişim için 4 adet SFP modül yuvası dikkati çekiyor. Kartın merkezinde yer alan Virtex 6 FPGA yongasının bir fan ile soğutulduğu görülüyor.

<p align="center">
  <img src="/virtex6_fpga.jpg"/>
</p>

Kartın üzerinde bulunan FPGA'nın tam modeli <code>XC6VLX365T-FF1156-1</code>. 364032 mantık hücresi, 14976 Kb BRAM, 576 DSP bloğu içeriyor.

Kartın beslemesi, birlikte gelen 24W adaptörle 12V DC girişinden veya PCI Express üzerinden sağlanabiliyor. Kartın alt kısmında yer alan DIP switch'ler üzerinden PCIe modu (x1, x4, x8) seçilebiliyor. x4 modunda PCIe 2.0, x8 modunda ise PCIe 1.0 olarak çalışabiliyor. Kartta 67 adet giriş/çıkış pini, 8 adet LED, 3 adet buton ve bir adet açma-kapama anahtarı yer alıyor. Ek olarak JTAG ve mini USB bağlantısı bulunuyor. Programlama için yalnızca JTAG kullanılırken, mini USB bağlantısı klon Arduino kartlarından bildiğimiz CH340 seri dönüştürücü entegresine bağlı.

<p align="center">
  <img src="/adapter.jpg"/>
</p>

Kartın üzerinde dahili programlayıcı yok. Bu sebeple ayrıca bir Xilinx Platform Cable cihazı sipariş ettim.

<p align="center">
  <img src="/platform_cable.jpg"/>
</p>

> :question: Peki bu şekilde beklenenden ucuza satılan bir kartın hikayesi ne olabilir? Bazen seri üretilmiş ve ömrünü tamamlamış bir ürünün içinden çıkan kartları Aliexpress gibi platformlarda satışa sunabiliyorlar. Kripto para madenciliği için kullanılan "ASIC miner" cihazlarının işe yaramaz hale geldiğinde, içlerindeki ZYNQ yongalı kartların satıldığını daha önce gördük (Doğrusunu söylemek gerekirse ben de bunlardan bir tane aldım, ilerleyen zamanda belki onu da inceleriz). Peki bu kart onlar gibi mi? Cevap muhtemelen hayır, çünkü kart üzerinde bunu hissettirecek bir ibare yer almıyor. Muhtemelen bu kart geliştirme ve demo amaçlı tasarlandı. Ayrıca gelen kartın kullanılmış gibi durmadığını da ekleyeyim. Ancak FPGA yongasının başka bir üründen çıkarılmış olması elbette olası.

Kartta keşke olsa diyeceğim şey, DDR bellek veya en azından SODIMM yuvası. Bunun haricinde kurcalamak için gayet yeterli görünüyor. Şimdi yüzleşmemiz gereken en büyük sorun 6 serisi Xilinx FPGA'ların Vivado tarafından desteklenmemesi.

Xilinx'in eski seri FPGA'ları için sunduğu ISE yazılımı ne yazık ki artık güncelleme almıyor. Bu nedenle güncel sistemlerde ISE'yi çalıştırmak hem zahmetli hem de problemli. Yayınlanan son ISE sürümü 14.7, Windows 10 ve yukarısı için desteklenmiyor. Fakat Xilinx, bu sistemler için sanal makine tabanlı bir çözüm sunmuş.

## ISE 14.7 VM Kurulumu

Xilinx web sayfasından ISE 14.7 VM sürümünü indirip <code>xsetup.exe</code> üzerinden kurulumu başlatıyorum ve ilk hatayı alıyorum. Hatada BIOS'da sanallaştırma ayarının açık olmadığı yazıyor.

<p align="center">
  <img src="/virtualization_error.png"/>
</p>

BIOS üzerinden sanallaştırma ayarını kontrol etmekte fayda var, fakat bu hata sanallaştırma ayarı açık olduğu halde de ortaya çıkabiliyor. Bu durumda kuruluma devam edebilmek için sanallaştırma kontrolünü atlatmak gerekiyor. Bunun için kurulum dizinindeki <code>bin/validate_virtualization_enabled.bat</code> betiğinin ikinci satırının başına "#" ifadesi ekliyoruz.

<p align="center">
  <img src="/mark_command.png"/>
</p>

<code>xsetup.exe</code>'yi yeniden çalıştırıp kurulumu başlattığımda, kullandığım VirtualBox sürümünün test edilmediğini belirten bir uyarı mesajı karşılıyor. Onaylayıp kuruluma devam ediyorum.

<p align="center">
  <img src="/untested_warning.png"/>
</p>

Sanal makine ile paylaşılan ortak klasör oluşturmak önemli, zira proje dosyalarını buradan kolaylıkla iki makine arasında transfer edebileceğiz.

<p align="center">
  <img src="/installation_dir.png"/>
</p>

Kurulum tamamlandıktan sonra başlat menüsüne eklenen Project Navigator'a tıklayıp sanal makineyi çalıştırıyoruz. VirtualBox penceresi açılıyor ve ardından sistem boot ediyor.

<p align="center">
  <img src="/ise_virtual_box.png"/>
</p>

> :pushpin: Varsayılan GTK teması resimde gösterilenden farklı. Ben ayarlardan değiştirdim.

Sanal makine kalıbında Oracle Linux işletim sistemi yüklü. Kernel sürümü 2.6 ve GNOME 2 masaüstü ortamıyla birlikte geliyor. Sistemin oldukça eski olduğunu söyleyebiliriz.

<p align="center">
  <img src="/system_about.png"/>
</p>

İşletim sisteminde, Xilinx araçlarının yanı sıra bazı ekstra uygulamalar bulunuyor. Bunların arasında ilginç şekilde webcam, multimedya ve CD/DVD yazılımları bile var.

<p align="center">
  <img src="/bloatware.png"/>
</p>

Kurulum tamamlandığına göre sırada bir proje oluşturup karta yüklemek var.

## Proje Oluşturma

Örnek proje olarak daha önce Nexys A7 ve Tang Nano 9k kartlarında da denediğimiz [Matrak](/posts/riscv-1) RISC-V işlemciyi kullanacağız. ISE üzerinde "New Project" butonuna basıyoruz. Ardından bizi proje oluşturma sihirbazı karşılıyor. Burada projenin adını ve dizinini seçiyoruz. Bu noktada projeyi, ana makine ile paylaşımlı klasörde açmak mantıklı olacaktır.

<p align="center">
  <img src="/new_project_wiz.png"/>
</p>

Sonraki ekranda kullanılacak FPGA platformununu seçmemiz isteniyor. Bizim kartımız tahmin edebileceğiniz gibi listede yer almıyor, bu nedenle bilgileri elle giriyoruz.

<p align="center">
  <img src="/fpga_specs.png"/>
</p>

Projeyi oluşturma adımları tamamlandı. Şimdi yeni oluşturduğumuz projeye kaynak dosyalarını ekleyeceğiz. Test edeceğimiz proje bir RISC-V işlemcili sistem içeriyor. Detaylarını incelemek isterseniz [önceki yazılara](/posts) göz atabilirsiniz. Kaynak dosyaları projeye eklemek için "Add Source" kısmına tıklamak yeterli. Proje kodları aşağıdadır.

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

<details>
<summary>memory.v: <mark>kodu göstermek için tıklayın</mark></summary>

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Main Memory Module

module memory (
    input              clk_i,
    input              wen_i,    // Yazma yetkilendirme girişi
    input      [3:0]   stb_i,    // Bayt seçim girişi
    input      [31:0]  addr_i,   // Adres girişi
    input      [31:0]  data_i,   // Yazılacak veri
    output     [31:0]  data_o    // Okunan veri (Asenkron çıkış)
);

    (* ram_style = "distributed" *)
    reg [31:0] mem [0:511];

    // Programı belleğe yükle
    initial begin
       $readmemh("program.mem", mem);
    end

    assign data_o = mem[addr_i[31:2]];

    always @(posedge clk_i) begin
        if (wen_i) begin
            if (stb_i[0]) mem[addr_i[31:2]][7:0]   <= data_i[7:0];
            if (stb_i[1]) mem[addr_i[31:2]][15:8]  <= data_i[15:8];
            if (stb_i[2]) mem[addr_i[31:2]][23:16] <= data_i[23:16];
            if (stb_i[3]) mem[addr_i[31:2]][31:24] <= data_i[31:24];
        end
    end

endmodule
```

</details>

<details>
<summary>gpio.v: <mark>kodu göstermek için tıklayın</mark></summary>

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// GPIO Output Module

module gpio (
   input                clk_i,
   input                rst_i,
   input                sel_i,   // Seçim sinyali
   input                wen_i,   // Yazma yetkilendirme
   input [31:0]         data_i,  // Veri girişi, işlemciden geliyor.
   output [31:0]        data_o,  // Veri çıkışı, işlemciye gidiyor.
   output reg [7:0]     gpio_o   // GPIO çıkış pinleri
);

   // Çevrebirim seçilmişse ve yazma etkin değilse (okuma etkin) çıkış kaydedicisinin değerini işlemciye gönder.
   assign data_o = (sel_i & !wen_i) ? {24'b0, gpio_o} : 32'b0;

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin // Çıkışlar sıfırlanıyor.
         gpio_o <= 8'b1111_1111;
      end else begin
         // Çevrebirim seçilmişse ve yazma etkinse işlemciden gelen değeri çıkış kaydedicisine aktar.
         if (sel_i & wen_i) begin
            gpio_o <= data_i[7:0];
         end
      end
   end

endmodule
```

</details>

<details>
<summary>uart.v: <mark>kodu göstermek için tıklayın</mark></summary>

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// UART TX Module

module uart (
   input                clk_i,
   input                rst_i,
   input                sel_i,      // Seçim sinyali
   input                wen_i,      // Yazma yetkilendirme
   input [31:0]         addr_i,     // Adres girişi, işlemciden geliyor.
   input [31:0]         data_i,     // Veri girişi, işlemciden geliyor.
   output [31:0]        data_o,     // Veri çıkışı, işlemciye gidiyor.
   output               uart_tx_o   // UART TX bağlantısı
);

   localparam UART_TRANSMIT_REG  = 4'h0;
   localparam UART_STATUS_REG    = 4'h4;

   wire done;

   // Kaydedici adresi çözümleniyor.
   wire tx_sel       = (UART_TRANSMIT_REG == addr_i[3:0]);
   wire status_sel   = (UART_STATUS_REG == addr_i[3:0]);

   // Gönderilecek veri yazılıyor. (gönderimi başlat)
   wire tx_en     = sel_i & wen_i & tx_sel;

   // Durum okunuyor.
   wire status_en = sel_i & status_sel;

   assign data_o  = status_en ? {30'b0, done} : 31'b0;

   transmitter t1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
      .tx_data_i(data_i[7:0]),
      .tx_en_i(tx_en),
      .tx_done_o(done),
      .tx_o(uart_tx_o)
   );

endmodule

module transmitter (
   input                clk_i,
   input                rst_i,
   input [7:0]          tx_data_i,
   input                tx_en_i,
   output reg           tx_done_o,
   output reg           tx_o
);

   localparam IDLE      = 2'b00;
   localparam START     = 2'b01;
   localparam TRANSMIT  = 2'b10;
   localparam DONE      = 2'b11;

   localparam CLKFREQ   = 50_000_000;
   localparam BAUD_RATE = 115200;

   localparam BAUD_DIV  = CLKFREQ/BAUD_RATE;

   reg [15:0] t_counter;
   reg [2:0] b_counter;

   reg [7:0] shr;

   reg [1:0] state;

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         state       <= IDLE;
         t_counter   <= 0;
         b_counter   <= 0;
         shr         <= 8'b0;
         tx_done_o   <= 1'b1;
         tx_o        <= 1'b1;
      end else begin
         case (state)
            IDLE : begin
               b_counter   <= 0;
               tx_done_o   <= 1'b1;
               tx_o        <= 1'b1;
               if (tx_en_i) begin
                  tx_o     <= 1'b0;
                  shr      <= tx_data_i;
                  state    <= START;
               end else begin
                  state    <= IDLE;
               end
            end
            START : begin
               tx_done_o   <= 1'b0;
               if (t_counter == BAUD_DIV-1) begin
                  t_counter   <= 0;
                  shr[7]      <= shr[0];
                  shr[6:0]    <= shr[7:1];
                  tx_o        <= shr[0];
                  state       <= TRANSMIT;
               end else begin
                  t_counter   <= t_counter + 1;
               end
            end
            TRANSMIT : begin
               tx_done_o   <= 1'b0;
               if (b_counter == 7) begin
                  if (t_counter == BAUD_DIV-1) begin
                     t_counter   <= 0;
                     b_counter   <= 0;
                     tx_o        <= 1'b1;
                     state       <= DONE;
                  end else begin
                     t_counter   <= t_counter + 1;
                  end
               end else begin
                  if (t_counter == BAUD_DIV-1) begin
                     t_counter   <= 0;
                     b_counter   <= b_counter + 1;
                     shr[7]      <= shr[0];
                     shr[6:0]    <= shr[7:1];
                     tx_o        <= shr[0];
                  end else begin
                     t_counter   <= t_counter + 1;
                  end
               end
            end
            DONE : begin
               if (t_counter == BAUD_DIV-1) begin
                  t_counter   <= 0;
                  tx_done_o   <= 1'b1;
                  state       <= IDLE;
               end else begin
                  t_counter   <= t_counter + 1;
               end
            end
            default : state <= IDLE;
         endcase
      end
   end

endmodule
```

</details>

<details>
<summary>clock_counter.v: <mark>kodu göstermek için tıklayın</mark></summary>

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Clock Counter Register

module clock_counter (
   input                clk_i,
   input                rst_i,
   input                sel_i,   // Seçim sinyali
   output [31:0]        data_o   // Veri çıkışı, işlemciye gidiyor.
);

   reg [31:0] counter_reg;

   assign data_o = sel_i ? counter_reg : 32'b0;

   always @(posedge clk_i, posedge rst_i) begin
      if (rst_i) begin
         counter_reg <= 32'b0;
      end else begin
         counter_reg <= counter_reg + 1;
      end
   end

endmodule
```

</details>

<details>
<summary>top.v: <mark>kodu göstermek için tıklayın</mark></summary>

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Top Module

module top (
   input             clk_i,
   input             rst_i,
   output [7:0]      gpio_o,
   output            uart_tx_o
);

   // İşlemci bağlantıları
   wire stall;
   wire wen;
   wire ren;
   wire [3:0] stb;
   wire [31:0] inst_addr;
   wire [31:0] data_addr;
   wire [31:0] wdata;
   wire [31:0] rdata;

   // Bellek bağlantıları
   wire [31:0] mem_addr;
   wire [31:0] mem_rdata;
   wire mem_wen;

   matrak mt1 (
      .clk_i(clk_i),
      .rst_i(!rst_i),
      .stall_i(stall),
      .inst_i(mem_rdata),
      .data_i(rdata),
      .wen_o(wen),
      .ren_o(ren),
      .stb_o(stb),
      .inst_addr_o(inst_addr),
      .data_addr_o(data_addr),
      .data_o(wdata)
   );

   memory me1 (
      .clk_i(clk_i),
      .wen_i(mem_wen),
      .stb_i(stb),
      .addr_i(mem_addr),
      .data_i(wdata),
      .data_o(mem_rdata)
   );

   // GPIO bağlantıları
   wire [31:0] gpio_rdata;
   wire gpio_request;

   gpio g1 (
      .clk_i(clk_i),
      .rst_i(!rst_i),
      .sel_i(gpio_request),
      .wen_i(wen),
      .data_i(wdata),
      .data_o(gpio_rdata),
      .gpio_o(gpio_o)
   );

   // UART bağlantıları
   wire uart_request;
   wire [31:0] uart_rdata;

   uart u1 (
      .clk_i(clk_i),
      .rst_i(!rst_i),
      .sel_i(uart_request),
      .wen_i(wen),
      .addr_i(data_addr),
      .data_i(wdata),
      .data_o(uart_rdata),
      .uart_tx_o(uart_tx_o)
   );

   // Saat sayacı bağlantıları
   wire clock_counter_request;
   wire [31:0] clock_counter_rdata;

   clock_counter cc1(
      .clk_i(clk_i),
      .rst_i(!rst_i),
      .sel_i(clock_counter_request),
      .data_o(clock_counter_rdata)
   );

   wire loadstore_request;
   wire peripheral_access;
   wire memory_ls_access;
   wire [31:0] periph_rdata;

   // Load Store isteği kontrol ediliyor.
   assign loadstore_request   = wen | ren;

   // Adres: 0x8XXXXXXX, çevrebirimlere yönlendiriliyor.
   assign peripheral_access   = loadstore_request & data_addr[31];

   // Okuma yazma isteği belleğe yönlendiriliyor.
   assign memory_ls_access    = loadstore_request & !peripheral_access;

   // Çevrebirimlerden okunan veri seçiliyor.
   assign periph_rdata        = gpio_request ? gpio_rdata : (uart_request ? uart_rdata :
                                 (clock_counter_request ? clock_counter_rdata : 32'b0));

   // Adrese göre seçilen çevrebirim belirleniyor.
   assign gpio_request           = peripheral_access & data_addr[12] & !data_addr[13]; // 0x80001000
   assign uart_request           = peripheral_access & data_addr[13] & !data_addr[12]; // 0x80002000
   assign clock_counter_request  = peripheral_access & data_addr[12] & data_addr[13];  // 0x80003000

   // Belleğe yazma isteği gönderiliyor.
   assign mem_wen    = wen & memory_ls_access;

   // Belleğe aktarılacak adres seçiliyor. (veri adresi veya buyruk adresi)
   assign mem_addr   = memory_ls_access ? data_addr : inst_addr;

   // İşlemciye gönderilecek veri seçiliyor. (bellekten veya çevrebirimlerinden okunan veri)
   assign rdata      = memory_ls_access ? mem_rdata : periph_rdata;

   // Bellek meşgulse işlemciyi durdur.
   assign stall      = memory_ls_access;

endmodule
```

</details>

<p></p>

Xilinx ISE, kısıt dosyası için .ucf formatını kullanıyor. Elimizdeki kart için aşağıdaki gibi bir kısıt dosyası oluşturabiliriz.

virtex6.ucf:

```
# Saat sinyali
NET "clk_i" LOC = H28 | IOSTANDARD = LVCMOS25;

NET "clk_i" TNM_NET = clk_i;
TIMESPEC TS_clk = PERIOD "clk_i" 10 ns;

# Reset butonu
NET "rst_i" LOC = A10 | IOSTANDARD = "LVCMOS25";

# UART çıkışı
NET uart_tx_o LOC = B8 | IOSTANDARD = "LVCMOS25";

# İşlemci çıkış pinleri
NET gpio_o<0>  LOC = AN18 | IOSTANDARD = "LVCMOS25";
NET gpio_o<1>  LOC = AN17 | IOSTANDARD = "LVCMOS25";
NET gpio_o<2>  LOC = AP17 | IOSTANDARD = "LVCMOS25";
NET gpio_o<3>  LOC = AP16 | IOSTANDARD = "LVCMOS25";
NET gpio_o<4>  LOC = AN15 | IOSTANDARD = "LVCMOS25";
NET gpio_o<5>  LOC = AP15 | IOSTANDARD = "LVCMOS25";
NET gpio_o<6>  LOC = AN14 | IOSTANDARD = "LVCMOS25";
NET gpio_o<7>  LOC = AP14 | IOSTANDARD = "LVCMOS25";
```

İşlemci içinde çalıştırmak için bir programa ihtiyacımız var. Bunun için diğer projelerde de kullandığım örneği seçtim.

knight.c:

```c
#include "matrak.h"

#define HIGH 0
#define LOW 1

void main(void) {

   while (1) {
      for (int i = 0; i < 8; i++) {
         gpio_write(i, HIGH);
         delay_ms(50);
         gpio_write(i, LOW);
      }
      for (int j = 7; j > -1; j--) {
         gpio_write(j, HIGH);
         delay_ms(50);
         gpio_write(j, LOW);
      }
   }
}
```

Kodu derleyip <code>program.mem</code> dosyasını oluşturuyoruz. Önceki bölümlerde bu konuyu ele aldığımız için burada ayrıca değinmedim.

<details>
<summary>program.mem: <mark>kodu göstermek için tıklayın</mark></summary>

```
00000093
00000113
00000193
00000213
00000293
00000313
00000393
00000413
00000493
00000513
00000593
00000613
00000693
00000713
00000793
00000813
00000893
00000913
00000993
00000a13
00000a93
00000b13
00000b93
00000c13
00000c93
00000d13
00000d93
00000e13
00000e93
00000f13
00000f93
00001137
80010113
00000513
00000593
1a8000ef
00000013
ffdff06f
fe010113
00812e23
02010413
fea42623
feb42423
fe842783
02078663
800017b7
0007a703
fec42783
00100693
00f697b3
00078693
800017b7
00d76733
00e7a023
02c0006f
800017b7
0007a703
fec42783
00100693
00f697b3
fff7c793
00078693
800017b7
00d77733
00e7a023
00000013
01c12403
02010113
00008067
fd010113
02812623
03010413
fca42e23
800037b7
0007a683
fdc42703
00070793
00179793
00e787b3
00679613
00c787b3
00279793
00e787b3
00279793
00e787b3
00479793
00f687b3
fef42623
00000013
800037b7
0007a783
fec42703
fee7eae3
00000013
00000013
02c12403
03010113
00008067
fd010113
02812623
03010413
fca42e23
fe042623
800027b7
fdc42703
00e7a023
0100006f
800027b7
0047a783
fef42623
fec42783
fe0788e3
00000013
00000013
02c12403
03010113
00008067
fd010113
02112623
02812423
03010413
fca42e23
fdc42783
fef42623
0200006f
fec42783
0007c783
00078513
f89ff0ef
fec42783
00178793
fef42623
fec42783
0007c783
fc079ee3
00000013
00000013
02c12083
02812403
03010113
00008067
fe010113
00112e23
00812c23
02010413
fe042623
0300006f
00000593
fec42503
e45ff0ef
06400513
eb9ff0ef
00100593
fec42503
e31ff0ef
fec42783
00178793
fef42623
fec42703
00700793
fce7d6e3
00700793
fef42423
0300006f
00000593
fe842503
e01ff0ef
06400513
e75ff0ef
00100593
fe842503
dedff0ef
fe842783
fff78793
fef42423
fe842783
fc07d8e3
f81ff06f
```
</details>

<p></p>

> :warning: <code>program.mem</code> dosyası, kaynak dosyalarının bulunduğu dizinde yer almalı ancak projeye eklenmemeli. Aksi halde bitstream aşamasında hata alabilirsiniz.

Kaynak dosyalarımızı ekledikten sonra sentez ve gerçekleme aşamalarına geçmeye hazırız. Üst modüle sağ tıklayıp "Implement Top Module" seçeneğine tıklayıp işlemlerin tamamlanmasını bekliyoruz. İşlem başarıyla tamamlanırsa "Generate Programming File"a tıklayıp bitstream oluşturabiliriz.

Proje, toplam 455040 kaydediciden yalnızca 225'ni ve 227520 LUT'dan yalnızca 1770 kadarını kullanıyor. Kullandığımız FPGA'nın bu proje için fazla büyük olduğunu söyleyebiliriz.

Şimdi sıra kartı programlamaya geldi. Sanal makine üzerinde çalıştığımız için bu kısım biraz kafa karıştırıcı olabilir. Programlayıcının yanında Windows sürücüleri geliyor. Fakat bu sürücüleri kurmaya gerek yok, tek yapılması gereken programlayıcıyı bilgisayara bağlayıp ardından sanal makineyi başlatmak. VirtualBox otomatik olarak programlayıcıyı misafir sisteme bağlıyor. Benim kullandığım DLC9LP kodlu programlayıcının sürücüleri sanal makine kalıbında yüklü geliyor. Sizin kullandığınız cihaz farklıysa, uçbirime <code>lsusb</code> yazarak cihazın tanınıp tanınmadığını kontrol edebilirsiniz.

ISE üzerinden "Configure Target Device"a tıklayıp IMPACT programını başlatıyoruz. Ekrandaki "Boundary Scan" seçeneğine çift tıklıyoruz. Açılan sekmeye sağ tıklayıp "Cable Setup" kısmına giriyoruz. Buradaki ekranda kullandığımız programlayıcının tipini seçmemiz gerekiyor. Benim durumumda Platform Cable USB/II seçilmesi gerekiyor.

<p align="center">
  <img src="/cable_setup.png"/>
</p>

Kablo seçimi tamamlandıktan sonra yine sağ tıklayıp "Cable Auto Connect"i seçiyoruz. Bu noktada bağlantının gerçekleşip gerçekleşmediğini kontrol etmek için konsol çıktısını okuyabilirsiniz. İşler yolunda gittiyse yine sağ tık menüsünden "Initalize Chain" seçeneğine tıklıyoruz, ardından "Assign New Configuration File" seçeneğine tıklayıp daha önce üretilen .bit uzantılı bitstream dosyasını seçiyoruz. Son olarak sağ tık menüsünden "Program" seçeneğine tıklayıp bitstream dosyasını FPGA'ya yüklüyoruz.

<p align="center">
  <img src="/impact.png"/>
</p>

Ve bu tanıdık projeyi Virtex 6 FPGA üzerinde başarıyla çalıştırmış oluyoruz.

<p align="center">
  <img src="/virtex_project.gif"/>
</p>
