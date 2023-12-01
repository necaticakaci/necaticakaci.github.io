---
title: "RISC-V İşlemci Tasarımı - Bölüm 5: Çevrebirimler ve C Kütüphanesi"
date: 2023-10-23
description: Bu bölümde GPIO ve UART gibi çevrebirimleri tasarlıyor ve işlemcimize bir kütüphane yazıyoruz.
math: true
---

İşlemcimizin tasarımını tamamlamışken en heyecanlı yerde bırakmak olmazdı. Bu yazıda bazı çevrebirimlerin tasarımına ve C ile programlar geliştirmeye değineceğiz. Ayrıca işlemcimizin fiziksel ortamda (FPGA) çalıştığına şahit olacağız.

## Belleğe Eşlemeli Çevrebirimleri

Belleğimiz 2 KiB boyutundaydı, 32-bit genişlikte bir adres alanı için gayet ufak. Peki boş kalan adres uzayını farklı amaçlar için kullansak nasıl olur? Güncel bilgisayar mimarilerinde, sıklıkla bellek haritasına eşlenmiş (memory mapped) çevrebirimlerine rastlarız. Çevrebirimleri sayesinde çeşitli girdi çıktı işlemlerini yönetebiliriz. Örneğin bir mikrodenetleyicin genel amaçlı giriş çıkış pinlerine LED bağlayıp, yakıp söndürebiliriz. Veya seri uçbirime "merhaba dünya" yazdırabiliriz. Bunların her biri bir çevrebirim vasıtasıyla mümkün olmaktadır.

Peki işlemci, çevrebirimleri ile nasıl anlaşır? Her çevrebirimini yönetmek için ek buyruklar eklemeyi düşünebilirdik, fakat çevrebirimlerinin çeşidi ve adedi arttıkça işlemcimiz oldukça karışmaya başlayacaktır. Daha iyi bir çözüm olarak bellek adres uzayının bazı kısımlarını çevrebirimleri ile iletişim kurmak için ayırabiliriz. Neticede gigabaytlarca bir alanımız var ve biz burada belleklerden bahsederken kilobaytları kullanıyoruz. Belleğe eşleme yöntemi ile tıpkı belleğe erişir gibi (load-store buyrukları ile) çevrebirimlerimizi yönetebilmekteyiz.

Bu bölümde 3 basit çevrebiriminin tasarımına bakacağız. Bunlar: 8 pin GPIO çıkış, 32-bit saat sayacı ve UART TX. Bu çevrebirimlerinin bellek haritasına yerleşimi aşağıdaki gibi olacak.

<p align="center">
  <img src="/memory-map.png"/>
</p>

<p align="center">
  İşlemcimizin bellek uzayı
</p>

## GPIO

GPIO deyimi aslında ayarlanabilir genel amaçlı giriş çıkış çevrebirimini tarif etse de biz gayet basit, 8 pinli sabit çıkışlı bir birim tasarlayacağız. Bu çıkış pinlerimiz ile LED, step motor ve röle gibi elektronik parçaları kontrol edebiliriz.

İşlemcimiz GPIO çevrebirimini <code>gpio_o</code> kaydedicisi üzerinden kontrol edecek. <code>gpio_o</code> kaydedicisi işlemci tarafından hem okunabilir hem de yazılabilir durumda olacak.

| Adres        | Kaydedici       | Bitler                         | İzinler      |
| ------------ | --------------- | ------------------------------ | ------------ |
| 0x80001000   | gpio_o          | boş[31:8], çıkış değeri[7:0]   | okuma/yazma  |

gpio.v:

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
         gpio_o <= 8'b0;
      end else begin
         // Çevrebirim seçilmişse ve yazma etkinse işlemciden gelen değeri çıkış kaydedicisine aktar.
         if (sel_i & wen_i) begin
            gpio_o <= data_i[7:0];
         end
      end
   end

endmodule
```

## Saat Sayacı

Bu birim her saat vuruşunda değeri 1 artan 32-bit genişlikte bir sayaç. Saat sayacını zamanlama gerektiren uygulamalarda kullanacağız.

| Adres        | Kaydedici       | Bitler                         | İzinler   |
| ------------ | --------------- | ------------------------------ | --------- |
| 0x80003000   | counter_reg     | saat sayacı[31:0]              | okuma     |

clock_counter.v:

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

## UART TX

Seri haberleşme işlemcimiz için oldukça önemli. Şimdilik sadece UART verici (transmitter) kısmı ile ilgileneceğiz. UART çevrebirimlerinde genelde baudrate değeri ayarlanabilir olur ama biz işi basit tutacağız. UART çevrebirimimiz sabit 115200 baudrate değerinde çalışacak.

| Adres        | Kaydedici       | Bitler                            | İzinler   |
| ------------ | --------------- | --------------------------------- | --------- |
| 0x80002000   | uart_transmit   | boş[31:8], gönderilecek veri[7:0] | yazma     |
| 0x80002004   | uart_status     | boş[31:1], hazır[0]               | okuma     |

UART ile veri göndermeden önce <code>0x80002004</code> adresindeki <code>uart_status</code> kaydedicisinin 0. bitini kontrol etmeliyiz. Bu bit 1 ise, UART veri göndermeye hazırdır. <code>0x80002000</code> adresindeki <code>uart_transmit</code> kaydedicisine göndermek istediğimiz karakteri yazmalıyız. Bu işlemin ardından UART çevrebirimi veri aktarımını başlatacaktır.

> :pushpin: Esasında UART modül tasarımı tek başına bölüm olmayı hak edecek kadar önemli bir konu. Fakat biz UART çevrebirimini şimdilik sadece işlemcimizin çalıştığını gözlemlemek için kullanacağız ve tasarım detaylarına değinmeyeceğiz.

uart.v:

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

## İşlemci ve Çevrebirimleri Bir Araya getirme

Çevrebirimlerimizi tasarladık. Şimdi işlemcimiz ile çevrebirimlerini birleştireceğiz. Önceki bölümlerde load ve store işlemleri yalnızca bellekten yapılabiliyordu. Artık işlemciden yazma veya okuma isteği geldiğinde adresi çözümleyip bellek, GPIO, UART ve saat sayacı çevrebirimlerinden birine yönlendirmemiz gerekiyor.

top.v:

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

   matrak mt1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
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

   // Bellek bağlantıları
   wire [31:0] mem_addr;
   wire [31:0] mem_rdata;
   wire mem_wen;

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
      .rst_i(rst_i),
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
      .rst_i(rst_i),
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
      .rst_i(rst_i),
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

> :pushpin: Üst modül kodunda görebileceğiniz gibi adresi, 32-bit yerine birkaç bit üzerinden kontrol ediyoruz. Bu bize donanımı basitleştirme imkanı sağlasa da, aynı çevrebirimine birden fazla adresle erişme durumunu ortaya çıkarıyor.

> :pushpin: Çevrebirim adresini (0x8XXXXXXX) kesin belirlemek için <code>loadstore_request & data_addr[31]</code> ifadesi yerine <code>loadstore_request & (data_addr[31:28] == 4'b1000)</code> yazmalıyız. Biz bu kadar yüksek adreslere başka amaçlar için erişmeyeceğiz, dolayısıyla herhangi bir değişiklik yapmamıza lüzum yok.

## C ile Programlama

Buraya kadar işlemcimizi assembly ile test ettik. Artık C programlama diline geçmeye hazırız. Öncelikle derleyici araçlarını (toolchain) bilgisayarımıza kurmalıyız. Açık kaynaklı derleyici araçlarını [RISC-V GNU Compiler Toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) GitHub reposundan edinebiliriz. Derleyici araçlarını kurmak için öncelikle bilgisayarınızda derlemeniz gerekiyor. Toolchain kurulumunun "Newlib" ve "Linux" olarak ayrıldığını görüyoruz. Linux işletim sistemi çalıştıran bir RISC-V işlemciniz varsa C kütüphanesi tercihinizi Linux'tan yana kullanmalısınız. Biz sıfırdan tasarladığımız "barebone" işlemcimiz için Newlib kütüphanesini seçmeliyiz. İşlemci mimarimiz de <code>-march rv32i</code> olarak seçilmeli.

> :pushpin: Bu kısım Linux işletim sistemi (Ubuntu) üzerinde çalıştığımız farzedilerek anlatılmıştır.

> :bulb: Derleme işlemi, yeni başlayanlar için meşakkatli gelebir. Uğraşmak istemeyenler [bu önceden derlenmiş](https://github.com/stnolting/riscv-gcc-prebuilt) (resmi olmayan) araçları edinebilirler.

Eğer önceden derlenmiş bir toolchain kullanmayı tercih ederseniz arşivden çıkarttığınız dizini Path'e eklemeniz gerekir. Uçbirime aşağıdaki komutları girerek toolchain'i kalıcı olarak Path'e ekleyebilirsiniz.

```
$ cd
$ nano .bashrc
```

Nano editöründe açılan .bashrc dosyasının uygun bir yerine <code>export PATH="/$HOME/kurulum_dizini/riscv32-unknown-elf/bin/:$PATH"</code> komutunu yapıştırın ve kaydedip çıkın.

Her şey doğru gittiyse uçbirime <code>riscv32-unknown-elf-gcc</code> komutu girdiğinizde uygulamanın çağrılmış olması gerekir.

Öyleyse Matrak işlemcimiz için ilk C kodumuzu yazmaya başlayalım. Şimdilik boş bir şablon hazırladık.

demo.c:

```c
void main(void) {

   while (1) {

   }
}
```

Bu C kodu RV32I mimarisi için derlenirse aşağıdaki assembly koduna dönüşecek.

```
main:
   addi    sp,sp,-16
   sw      s0,12(sp)
   addi    s0,sp,16
.L2:
   j       .L2
```

> :bulb: Yüksek seviyeli programlama dili ile hazırladığınız kodların assembly karşılıklarını incelemek için [Compiler Explorer](https://godbolt.org/) sitesini ziyaret edebilirsiniz.

Fakat bu kodu doğrudan işlemcimize yükleyemeyiz. Evvelinde kaydedici değerlerini sıfırlamak ve stack pointer (sp) kaydedicisine değer yüklemek gibi bazı başlangıç ayarlamalarını gerçekleştirmemiz gerekiyor. Bunu crt0.s assembly kodu ile halledeceğiz. crt0 ilk olarak başlangıç rutinlerini çalıştıracak ve ardından C kodumuzdaki main etiketine dallanacak.

Hatırlarsanız benzetim ortamında kaydedici dosyamızın başlangıç değerleri belirsiz olarak gözüküyordu. Bu yüzden ilk olarak tüm kaydedicilere 0 değerini yüklüyoruz. Daha sonra stack pointer kaydedicimize (x2) belleğimizin en yüksek adresini yüklüyoruz. Bu işlemlerin ardından main fonksiyonuna atlayabiliriz.

crt0.s:

```
.section .text
.global _start

_start:
   # Kaydedicileri sıfırla
   mv  x1, x0
   mv  x2, x0
   mv  x3, x0
   mv  x4, x0
   mv  x5, x0
   mv  x6, x0
   mv  x7, x0
   mv  x8, x0
   mv  x9, x0
   mv x10, x0
   mv x11, x0
   mv x12, x0
   mv x13, x0
   mv x14, x0
   mv x15, x0
   mv x16, x0
   mv x17, x0
   mv x18, x0
   mv x19, x0
   mv x20, x0
   mv x21, x0
   mv x22, x0
   mv x23, x0
   mv x24, x0
   mv x25, x0
   mv x26, x0
   mv x27, x0
   mv x28, x0
   mv x29, x0
   mv x30, x0
   mv x31, x0

   # Stack değerini yükle
   la x2, 0x800

# Main fonksiyonuna atla
jump_main:
   addi a0, x0, 0
   addi a1, x0, 0
   jal x1, main

# Yürütmeyi durdur
sleep_loop:
   nop
   j sleep_loop
```

> :pushpin: İşlemcimiz bir şekilde <code>sleep_loop</code> etiketine atlarsa sıfırlanıncaya kadar buradan çıkamaz. Bu, Motorola 6800 işlemcilerde iki illegal buyruğun yürütülmesiyle ortaya çıkan "[halt and catch fire](https://en.wikipedia.org/wiki/Halt_and_Catch_Fire_(computing))" (durdur ve yak) durumuna benziyor. Bu durum meydana gelirse işlemci sıfırlanıncaya kadar takılı kalıyormuş.

Kodlarımız hazır. Peki derleyici, buyrukları hangi adreslere yerleştirecek ve ne kadar belleğimizin olduğunu nereden bilecek? Burada linker script devreye giriyor. Linker script ile bellek boyutumuzu, program (text) ve verinin (data) nasıl yerleşeceği gibi bilgileri derleyiciye aktarıyoruz.

mem.ld:

```
OUTPUT_ARCH( "riscv" )

MEMORY
{
  RAM : ORIGIN = 0x00000000, LENGTH = 0x800
}

SECTIONS
{
  .text : {
    . = ALIGN(4);
    *(.text)
    *(.text.*)
    . = ALIGN(4);
  } > RAM

  .rodata : {
    . = ALIGN(4);
    *(.srodata)
    *(.srodata.*)
    *(.rodata);
    *(.rodata.*)
    . = ALIGN(4);
  } > RAM

  .data : {
    . = ALIGN(4);
    *(.sdata)
    *(.sdata.*)
    *(.data);
    *(.data.*)
    . = ALIGN(4);
  } > RAM
}
```

Artık derleme işlemine geçebiliriz. Öncelikle crt0.s assembly kodunu derliyoruz. crt0.o adında bir obje dosyası oluşacak. Bunu, bir sonraki adımda C kodumuz ile birleştireceğiz.

```
$ riscv32-unknown-elf-gcc -march=rv32i -mabi=ilp32 -nostdlib -Wl,-T,mem.ld -c crt0.s -o crt0.o
```

Bu işlemin ardından C kodumuz ve crt0.o dosyasından [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) (Executable Linkable Format) dosyası elde edeceğiz.

```
$ riscv32-unknown-elf-gcc -march=rv32i -mabi=ilp32 -nostdlib -Wl,-T,mem.ld crt0.o demo.c -o demo.elf
```

Üretilen ELF dosyasını işlemcimize yükleyebilmek için ikilik tabanda makine koduna dönüştürüyoruz.

```
$ riscv32-unknown-elf-objcopy -O binary demo.elf demo.bin
```

Aslına bakarsanız bin uzantılı dosya, belleğe yazılmak için hazır ancak biz belleğimizi program.mem dosyasına yazdığımız onaltılık tabandaki kod ile programlıyoruz. Bin uzantılı dosyayı belleğe yüklemeden önce onaltılık tabana çevirmemiz gerekiyor. Bu işlemi gerçekleştirmek için üçüncü parti bir araç kullanabilir veya kendimiz bir tane yazabiliriz.

Neyse ki SiFive şirketinin bu işlemi gerçekleştirmek için yazmış olduğu [bin2hex](https://github.com/sifive/freedom-elf2hex/blob/master/util/freedom-bin2hex.c) aracı var. Ancak bu araç C ile yazılmış, pratik olarak kullanmak için bunu Python scripti haline dönüştürmeliyiz.

bin2hex.py:

```python
# Based on: https://github.com/sifive/freedom-elf2hex/blob/master/util/freedom-bin2hex.c

import sys

def help():
    print("Convert a binary file to a format that can be read in verilog via $readmemh().")
    print("By default read from stdin and write to stdout using a bit width of 8.")
    print("\n")
    print("python bin2hex.py [--help|-h] [--bit-width|-w INT] [--input|-i BIN] [--output|-o HEX]")

def dump(output_stream, byte_width, byte_array):
    byte_width = int(byte_width)
    for i in range(byte_width - 1, -1, -1):
        output_stream.write(f'{byte_array[i]:02x}')
    output_stream.write('\n')

def main():
    ARRAY_SIZE = 1024
    bit_width = 32
    input_stream = sys.stdin.buffer
    output_stream = sys.stdout

    for i in range(len(sys.argv)):
        if sys.argv[i] in ["--help", "-h"]:
            help()
            return
        if sys.argv[i] in ["--bit-width", "-w"]:
            if i + 1 == len(sys.argv):
                sys.stderr.write("No arg for --bit-width|-w option.\n")
                return 1
            bit_width = int(sys.argv[i + 1])
        if sys.argv[i] in ["--input", "-i"]:
            if i + 1 == len(sys.argv):
                sys.stderr.write("No arg for --input|-i option.\n")
                return 1
            input_stream = open(sys.argv[i + 1], 'rb')
        if sys.argv[i] in ["--output", "-o"]:
            if i + 1 == len(sys.argv):
                sys.stderr.write("No arg for --output|-o option.\n")
                return 1
            output_stream = open(sys.argv[i + 1], 'w')

    if bit_width < 8:
        sys.stderr.write("Bit width cannot be negative or less than 8.\n")
        return 2
    if bit_width % 8 != 0:
        sys.stderr.write("Cannot handle non-multiple-of-8 bit width yet.\n")
        return 2
    if bit_width > (ARRAY_SIZE * 8):
        sys.stderr.write("Bit width is out of range (max supported is 8192).\n")
        return 3

    byte_width = bit_width / 8
    byte_value = 0
    byte_count = 0
    byte_array = bytearray([0] * ARRAY_SIZE)

    while True:
        byte_value = input_stream.read(1)
        if not byte_value:
            break
        byte_array[byte_count] = ord(byte_value)
        byte_count += 1
        if byte_count == byte_width:
            byte_count = 0
            dump(output_stream, byte_width, byte_array)
            byte_array = bytearray([0] * ARRAY_SIZE)

    if byte_count > 0:
        dump(output_stream, byte_width, byte_array)

if __name__ == "__main__":
    main()
```

Şimdi kodumuzu onaltılık tabana çevirebiliriz.

```
$ python3 bin2hex.py -i demo.bin -o demo.hex
```

Derleme komutlarını uçbirime tek tek girmek zahmetli olacağından işleri otomatize etmek için bir Makefile dosyası hazırlayabiliriz.

makefile:

```makefile
CC = riscv32-unknown-elf-gcc
OBJCOPY = riscv32-unknown-elf-objcopy
CFLAGS = -march=rv32i -mabi=ilp32 -nostdlib -Wl,-T,mem.ld

all: demo.hex

crt0.o: crt0.s
	$(CC) $(CFLAGS) -c crt0.s -o crt0.o

demo.elf: crt0.o demo.c
	$(CC) $(CFLAGS) crt0.o demo.c -o demo.elf

demo.bin: demo.elf
	$(OBJCOPY) -O binary demo.elf demo.bin

demo.hex: demo.bin
	python3 bin2hex.py -i demo.bin -o demo.hex

clean:
	rm -f crt0.o demo.elf demo.bin demo.hex
```

Şimdi demo.hex dosyasını oluşturmak için tek yapmamız gereken, uçbirime <code>make all</code> veya <code>make demo.hex</code> komutunu girmek.

## Matrak Kütüphanesi

Sıra geldi çevrebirimlerimiz için bir sürücü kütüphanesi yazmaya. Kaydedici adreslerimizi gösteren pointer'ları struct içinde tanımlıyoruz. Kütüphanemiz birkaç fonksiyondan ibaret olacak.

<code>gpio_write()</code>: <code>gpio_pin</code> argümanında verilen pini, <code>value</code> argümanında verilen değere göre 1 ya da 0 yapar.

<code>delay_ms()</code>: ms türünden verilen süre boyunca işlemcinin beklemesini sağlar.

<code>put_char()</code>: verilen karakteri UART üzerinden iletir.

<code>put_str()</code>: verilen karakter dizisinindeki elemanları <code>put_char()</code> fonksiyonunu çağırarak UART üzerinden iletir.

matrak.h:

```c
// Matrak Çevrebirim Kütüphanesi

#include <stdint.h>

#define GPIO0_BASE (0x80001000U)
#define GPIO0 ((GPIO*) GPIO0_BASE)

#define UART0_BASE (0x80002000U)
#define UART0 ((UART*) UART0_BASE)

#define CYC_C0_BASE (0x80003000U)
#define CYC_C0 ((CYC_C*) CYC_C0_BASE)

#define CLK_FREQ 50000000

typedef struct {
  volatile uint32_t gpio_output;  // 0x80001000 (RW) OUTPUT REGISTER
} GPIO;

typedef struct {
  volatile uint32_t uart_transmit;  // 0x80002000 (W) TX DATA REGISTER
  volatile uint32_t uart_status;    // 0x80002004 (R) TX STATUS REGISTER
} UART;

typedef struct {
  volatile uint32_t clock_counter;  // 0x80003000 (R) CLOCK COUNTER REGISTER
} CYC_C;

void gpio_write(int gpio_pin, int value) {
   if (value) {
      GPIO0->gpio_output |= (1<<gpio_pin);
   }
   else {
      GPIO0->gpio_output &= ~(1<<gpio_pin);
   }
}

void delay_ms(int time) {
   uint32_t clk_value = CYC_C0->clock_counter + (time * CLK_FREQ) / 1000;
   while (CYC_C0->clock_counter < clk_value);
}

void put_char(int c) {
   uint32_t done = 0;
   UART0->uart_transmit = c;
   while (done == 0) {
      done = UART0->uart_status;
   }
}

void put_str(const char* s) {
   for (const char* p=s; *p; ++p) {
      put_char(*p);
   }
}
```

Kütüphanemiz hazır. Öyleyse kütüphanemizi kullanarak bir led yakma söndürme uygulaması yapalım. 4 numaralı çıkış pinini, 500 ms aralıkla 1 ve 0 arasında döndürüyoruz.

blink.c:

```c
#include "matrak.h"

#define LED_PIN 4
#define HIGH 1
#define LOW 0

void main(void) {

   while (1) {
      gpio_write(LED_PIN, HIGH);
      delay_ms(500);
      gpio_write(LED_PIN, LOW);
      delay_ms(500);
   }
}
```

Oldukça basit değil mi? Belki biraz daha göze hoş gelen bir şeyler yapabiliriz. İşlemcimizin nasıl çalıştığı görmek için sabırsızlanıyoruz ancak yapacak son bir işimiz kaldı; FPGA ortamımızı hazırlamalıyız.

## FPGA Üzerinde Çalıştırma

Matrak işlemcimizi test etmek için Nexys A7 FPGA kartını seçtik. Bu kartın merkezinde Artix 7 (XC7A100T-1CSG324C) FPGA yongası yer alıyor. Kartta bulunan bazı bileşenler ise şöyle:

<p align="center">
  <img src="/nexys-a7.png"/>
</p>

<p align="center">
  Nexys A7 FPGA
</p>

* 3-eksen ivmeölçer
* PDM mikrofon
* PWM ses çıkışı
* Sıcaklık sensörü
* 2 adet dört dijit yedi segment ekran
* USB HID
* 16 anahtar
* 16 LED
* 2 RGB LED
* 12-bit VGA çıkışı
* 100 Mhz kristal osilatör

Kartın üzerinde çok fazla bileşen var. Biz şimdilik sadece LED'lerle ilgileneceğiz.

Vivado üzerinde yeni proje oluşturarak başlıyoruz. Açılan pencereden projemize bir isim verip ilerliyoruz. Bir sonraki pencede projenin kaynak dosyaları isteniyor. Burada işlemcimizin kaynak dosyalarını yüklüyoruz. Bu dosyalar; matrak.v, memory.v, program.mem, gpio.v, uart.v, clock_counter.v, top.v, top_tb.v ve wrapper.v

> :pushpin: Matrak işlemcimiz, Artix 7 FPGA üzerinde 100 Mhz frekansında çalışamıyor. Bu yüzden ilerleyen kısımda "Clocking Wizard" IP ile saat frekansını 50 Mhz değerine düşüreceğiz. Bunun için wrapper.v adı altında top.v modülünün üzerinde yer alan yeni bir üst modül hazırladık.

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
         gpio_o <= 8'b0;
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

   matrak mt1 (
      .clk_i(clk_i),
      .rst_i(rst_i),
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

   // Bellek bağlantıları
   wire [31:0] mem_addr;
   wire [31:0] mem_rdata;
   wire mem_wen;

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
      .rst_i(rst_i),
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
      .rst_i(rst_i),
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
      .rst_i(rst_i),
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

<details>
<summary>top_tb.v: <mark>kodu göstermek için tıklayın</mark></summary>

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Top Module Testbench

module top_tb ();

   reg tb_clk_i;
   reg tb_rst_i;
   wire [7:0] tb_gpio_o;
   wire tb_uart_tx_o;

   top t1 (
      .clk_i(tb_clk_i),
      .rst_i(tb_rst_i),
      .gpio_o(tb_gpio_o),
      .uart_tx_o(tb_uart_tx_o)
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

</details>

<details>
<summary>wrapper.v: <mark>kodu göstermek için tıklayın</mark></summary>

```verilog
// Matrak M10 RV32I RISC-V Processor
// Gülpare II Architechture 2023
// Nexys A7 Wrapper Module

module wrapper (
   input             clk_100_i,
   input             rst_i,
   output [7:0]      gpio_o,
   output            uart_tx_o
);

   wire clk_50;

   clk_50mhz clk1 (
      .clk_out1(clk_50),
      .clk_in1(clk_100_i)
   );

   top t1 (
      .clk_i(clk_50),
      .rst_i(!rst_i),
      .gpio_o(gpio_o),
      .uart_tx_o(uart_tx_o)
   );

endmodule
```

</details>

<p></p>

Ardından bizden bir "constraint" dosyası eklememiz isteniyor. Bu dosya üst modülümüz ile FPGA'nın fiziksel pinleri arasındaki bağlantıları belirtecek. 100 Mhz kristal osilatör girişini wrapper modülümüzün <code>clk_100_i</code> girişine, reset butonunu wrapper modülümüzün <code>rst_i</code> girişine bağlıyoruz. FPGA kartının ilk 8 LED'ini <code>gpio_o</code> çıkışına bağlıyoruz. İşlemcimizin <code>uart_tx_o</code> çıkışını FPGA kartının FTDI FT2232 USB-UART köprü yongasının RXD girişine bağlıyoruz.

> :pushpin: Nexys A7 kartı USB üzerinden programlanabildiği gibi aynı USB üzerinden seri haberleşmeyi de destekliyor. Eğer bu mümkün olmasaydı UART çıkışımızı, FPGA kartının dışarıdan erişebileceğimiz pinlerine yönlendirip bu pinlerden USB-TTL dönüştürücü bağlamamız gerekirdi.

<details>
<summary>Nexys-A7-100T-Master.xdc: <mark>kodu göstermek için tıklayın</mark></summary>

```
## This file is a general .xdc for the Nexys A7-100T
## To use it in a project:
## - uncomment the lines corresponding to used pins
## - rename the used ports (in each line, after get_ports) according to the top level signal names in the project

## Clock signal
set_property -dict { PACKAGE_PIN E3    IOSTANDARD LVCMOS33 } [get_ports { clk_100_i }]; #IO_L12P_T1_MRCC_35 Sch=clk100mhz
create_clock -add -name sys_clk_pin -period 10.00 -waveform {0 5} [get_ports {clk_100_i}];


##Switches
#set_property -dict { PACKAGE_PIN J15   IOSTANDARD LVCMOS33 } [get_ports { SW[0] }]; #IO_L24N_T3_RS0_15 Sch=sw[0]
#set_property -dict { PACKAGE_PIN L16   IOSTANDARD LVCMOS33 } [get_ports { SW[1] }]; #IO_L3N_T0_DQS_EMCCLK_14 Sch=sw[1]
#set_property -dict { PACKAGE_PIN M13   IOSTANDARD LVCMOS33 } [get_ports { SW[2] }]; #IO_L6N_T0_D08_VREF_14 Sch=sw[2]
#set_property -dict { PACKAGE_PIN R15   IOSTANDARD LVCMOS33 } [get_ports { SW[3] }]; #IO_L13N_T2_MRCC_14 Sch=sw[3]
#set_property -dict { PACKAGE_PIN R17   IOSTANDARD LVCMOS33 } [get_ports { SW[4] }]; #IO_L12N_T1_MRCC_14 Sch=sw[4]
#set_property -dict { PACKAGE_PIN T18   IOSTANDARD LVCMOS33 } [get_ports { SW[5] }]; #IO_L7N_T1_D10_14 Sch=sw[5]
#set_property -dict { PACKAGE_PIN U18   IOSTANDARD LVCMOS33 } [get_ports { SW[6] }]; #IO_L17N_T2_A13_D29_14 Sch=sw[6]
#set_property -dict { PACKAGE_PIN R13   IOSTANDARD LVCMOS33 } [get_ports { SW[7] }]; #IO_L5N_T0_D07_14 Sch=sw[7]
#set_property -dict { PACKAGE_PIN T8    IOSTANDARD LVCMOS18 } [get_ports { SW[8] }]; #IO_L24N_T3_34 Sch=sw[8]
#set_property -dict { PACKAGE_PIN U8    IOSTANDARD LVCMOS18 } [get_ports { SW[9] }]; #IO_25_34 Sch=sw[9]
#set_property -dict { PACKAGE_PIN R16   IOSTANDARD LVCMOS33 } [get_ports { SW[10] }]; #IO_L15P_T2_DQS_RDWR_B_14 Sch=sw[10]
#set_property -dict { PACKAGE_PIN T13   IOSTANDARD LVCMOS33 } [get_ports { SW[11] }]; #IO_L23P_T3_A03_D19_14 Sch=sw[11]
#set_property -dict { PACKAGE_PIN H6    IOSTANDARD LVCMOS33 } [get_ports { SW[12] }]; #IO_L24P_T3_35 Sch=sw[12]
#set_property -dict { PACKAGE_PIN U12   IOSTANDARD LVCMOS33 } [get_ports { SW[13] }]; #IO_L20P_T3_A08_D24_14 Sch=sw[13]
#set_property -dict { PACKAGE_PIN U11   IOSTANDARD LVCMOS33 } [get_ports { SW[14] }]; #IO_L19N_T3_A09_D25_VREF_14 Sch=sw[14]
#set_property -dict { PACKAGE_PIN V10   IOSTANDARD LVCMOS33 } [get_ports { SW[15] }]; #IO_L21P_T3_DQS_14 Sch=sw[15]

## LEDs
set_property -dict { PACKAGE_PIN H17   IOSTANDARD LVCMOS33 } [get_ports { gpio_o[0] }]; #IO_L18P_T2_A24_15 Sch=led[0]
set_property -dict { PACKAGE_PIN K15   IOSTANDARD LVCMOS33 } [get_ports { gpio_o[1] }]; #IO_L24P_T3_RS1_15 Sch=led[1]
set_property -dict { PACKAGE_PIN J13   IOSTANDARD LVCMOS33 } [get_ports { gpio_o[2] }]; #IO_L17N_T2_A25_15 Sch=led[2]
set_property -dict { PACKAGE_PIN N14   IOSTANDARD LVCMOS33 } [get_ports { gpio_o[3] }]; #IO_L8P_T1_D11_14 Sch=led[3]
set_property -dict { PACKAGE_PIN R18   IOSTANDARD LVCMOS33 } [get_ports { gpio_o[4] }]; #IO_L7P_T1_D09_14 Sch=led[4]
set_property -dict { PACKAGE_PIN V17   IOSTANDARD LVCMOS33 } [get_ports { gpio_o[5] }]; #IO_L18N_T2_A11_D27_14 Sch=led[5]
set_property -dict { PACKAGE_PIN U17   IOSTANDARD LVCMOS33 } [get_ports { gpio_o[6] }]; #IO_L17P_T2_A14_D30_14 Sch=led[6]
set_property -dict { PACKAGE_PIN U16   IOSTANDARD LVCMOS33 } [get_ports { gpio_o[7] }]; #IO_L18P_T2_A12_D28_14 Sch=led[7]
#set_property -dict { PACKAGE_PIN V16   IOSTANDARD LVCMOS33 } [get_ports { LED[8] }]; #IO_L16N_T2_A15_D31_14 Sch=led[8]
#set_property -dict { PACKAGE_PIN T15   IOSTANDARD LVCMOS33 } [get_ports { LED[9] }]; #IO_L14N_T2_SRCC_14 Sch=led[9]
#set_property -dict { PACKAGE_PIN U14   IOSTANDARD LVCMOS33 } [get_ports { LED[10] }]; #IO_L22P_T3_A05_D21_14 Sch=led[10]
#set_property -dict { PACKAGE_PIN T16   IOSTANDARD LVCMOS33 } [get_ports { LED[11] }]; #IO_L15N_T2_DQS_DOUT_CSO_B_14 Sch=led[11]
#set_property -dict { PACKAGE_PIN V15   IOSTANDARD LVCMOS33 } [get_ports { LED[12] }]; #IO_L16P_T2_CSI_B_14 Sch=led[12]
#set_property -dict { PACKAGE_PIN V14   IOSTANDARD LVCMOS33 } [get_ports { LED[13] }]; #IO_L22N_T3_A04_D20_14 Sch=led[13]
#set_property -dict { PACKAGE_PIN V12   IOSTANDARD LVCMOS33 } [get_ports { LED[14] }]; #IO_L20N_T3_A07_D23_14 Sch=led[14]
#set_property -dict { PACKAGE_PIN V11   IOSTANDARD LVCMOS33 } [get_ports { LED[15] }]; #IO_L21N_T3_DQS_A06_D22_14 Sch=led[15]

## RGB LEDs
#set_property -dict { PACKAGE_PIN R12   IOSTANDARD LVCMOS33 } [get_ports { LED16_B }]; #IO_L5P_T0_D06_14 Sch=led16_b
#set_property -dict { PACKAGE_PIN M16   IOSTANDARD LVCMOS33 } [get_ports { LED16_G }]; #IO_L10P_T1_D14_14 Sch=led16_g
#set_property -dict { PACKAGE_PIN N15   IOSTANDARD LVCMOS33 } [get_ports { LED16_R }]; #IO_L11P_T1_SRCC_14 Sch=led16_r
#set_property -dict { PACKAGE_PIN G14   IOSTANDARD LVCMOS33 } [get_ports { LED17_B }]; #IO_L15N_T2_DQS_ADV_B_15 Sch=led17_b
#set_property -dict { PACKAGE_PIN R11   IOSTANDARD LVCMOS33 } [get_ports { LED17_G }]; #IO_0_14 Sch=led17_g
#set_property -dict { PACKAGE_PIN N16   IOSTANDARD LVCMOS33 } [get_ports { LED17_R }]; #IO_L11N_T1_SRCC_14 Sch=led17_r

##7 segment display
#set_property -dict { PACKAGE_PIN T10   IOSTANDARD LVCMOS33 } [get_ports { CA }]; #IO_L24N_T3_A00_D16_14 Sch=ca
#set_property -dict { PACKAGE_PIN R10   IOSTANDARD LVCMOS33 } [get_ports { CB }]; #IO_25_14 Sch=cb
#set_property -dict { PACKAGE_PIN K16   IOSTANDARD LVCMOS33 } [get_ports { CC }]; #IO_25_15 Sch=cc
#set_property -dict { PACKAGE_PIN K13   IOSTANDARD LVCMOS33 } [get_ports { CD }]; #IO_L17P_T2_A26_15 Sch=cd
#set_property -dict { PACKAGE_PIN P15   IOSTANDARD LVCMOS33 } [get_ports { CE }]; #IO_L13P_T2_MRCC_14 Sch=ce
#set_property -dict { PACKAGE_PIN T11   IOSTANDARD LVCMOS33 } [get_ports { CF }]; #IO_L19P_T3_A10_D26_14 Sch=cf
#set_property -dict { PACKAGE_PIN L18   IOSTANDARD LVCMOS33 } [get_ports { CG }]; #IO_L4P_T0_D04_14 Sch=cg
#set_property -dict { PACKAGE_PIN H15   IOSTANDARD LVCMOS33 } [get_ports { DP }]; #IO_L19N_T3_A21_VREF_15 Sch=dp
#set_property -dict { PACKAGE_PIN J17   IOSTANDARD LVCMOS33 } [get_ports { AN[0] }]; #IO_L23P_T3_FOE_B_15 Sch=an[0]
#set_property -dict { PACKAGE_PIN J18   IOSTANDARD LVCMOS33 } [get_ports { AN[1] }]; #IO_L23N_T3_FWE_B_15 Sch=an[1]
#set_property -dict { PACKAGE_PIN T9    IOSTANDARD LVCMOS33 } [get_ports { AN[2] }]; #IO_L24P_T3_A01_D17_14 Sch=an[2]
#set_property -dict { PACKAGE_PIN J14   IOSTANDARD LVCMOS33 } [get_ports { AN[3] }]; #IO_L19P_T3_A22_15 Sch=an[3]
#set_property -dict { PACKAGE_PIN P14   IOSTANDARD LVCMOS33 } [get_ports { AN[4] }]; #IO_L8N_T1_D12_14 Sch=an[4]
#set_property -dict { PACKAGE_PIN T14   IOSTANDARD LVCMOS33 } [get_ports { AN[5] }]; #IO_L14P_T2_SRCC_14 Sch=an[5]
#set_property -dict { PACKAGE_PIN K2    IOSTANDARD LVCMOS33 } [get_ports { AN[6] }]; #IO_L23P_T3_35 Sch=an[6]
#set_property -dict { PACKAGE_PIN U13   IOSTANDARD LVCMOS33 } [get_ports { AN[7] }]; #IO_L23N_T3_A02_D18_14 Sch=an[7]

##CPU Reset Button
set_property -dict { PACKAGE_PIN C12   IOSTANDARD LVCMOS33 } [get_ports { rst_i }]; #IO_L3P_T0_DQS_AD1P_15 Sch=cpu_resetn

##Buttons
#set_property -dict { PACKAGE_PIN N17   IOSTANDARD LVCMOS33 } [get_ports { BTNC }]; #IO_L9P_T1_DQS_14 Sch=btnc
#set_property -dict { PACKAGE_PIN M18   IOSTANDARD LVCMOS33 } [get_ports { BTNU }]; #IO_L4N_T0_D05_14 Sch=btnu
#set_property -dict { PACKAGE_PIN P17   IOSTANDARD LVCMOS33 } [get_ports { BTNL }]; #IO_L12P_T1_MRCC_14 Sch=btnl
#set_property -dict { PACKAGE_PIN M17   IOSTANDARD LVCMOS33 } [get_ports { BTNR }]; #IO_L10N_T1_D15_14 Sch=btnr
#set_property -dict { PACKAGE_PIN P18   IOSTANDARD LVCMOS33 } [get_ports { BTND }]; #IO_L9N_T1_DQS_D13_14 Sch=btnd


##Pmod Headers
##Pmod Header JA
#set_property -dict { PACKAGE_PIN C17   IOSTANDARD LVCMOS33 } [get_ports { JA[1] }]; #IO_L20N_T3_A19_15 Sch=ja[1]
#set_property -dict { PACKAGE_PIN D18   IOSTANDARD LVCMOS33 } [get_ports { JA[2] }]; #IO_L21N_T3_DQS_A18_15 Sch=ja[2]
#set_property -dict { PACKAGE_PIN E18   IOSTANDARD LVCMOS33 } [get_ports { JA[3] }]; #IO_L21P_T3_DQS_15 Sch=ja[3]
#set_property -dict { PACKAGE_PIN G17   IOSTANDARD LVCMOS33 } [get_ports { JA[4] }]; #IO_L18N_T2_A23_15 Sch=ja[4]
#set_property -dict { PACKAGE_PIN D17   IOSTANDARD LVCMOS33 } [get_ports { JA[7] }]; #IO_L16N_T2_A27_15 Sch=ja[7]
#set_property -dict { PACKAGE_PIN E17   IOSTANDARD LVCMOS33 } [get_ports { JA[8] }]; #IO_L16P_T2_A28_15 Sch=ja[8]
#set_property -dict { PACKAGE_PIN F18   IOSTANDARD LVCMOS33 } [get_ports { JA[9] }]; #IO_L22N_T3_A16_15 Sch=ja[9]
#set_property -dict { PACKAGE_PIN G18   IOSTANDARD LVCMOS33 } [get_ports { JA[10] }]; #IO_L22P_T3_A17_15 Sch=ja[10]

##Pmod Header JB
#set_property -dict { PACKAGE_PIN D14   IOSTANDARD LVCMOS33 } [get_ports { JB[1] }]; #IO_L1P_T0_AD0P_15 Sch=jb[1]
#set_property -dict { PACKAGE_PIN F16   IOSTANDARD LVCMOS33 } [get_ports { JB[2] }]; #IO_L14N_T2_SRCC_15 Sch=jb[2]
#set_property -dict { PACKAGE_PIN G16   IOSTANDARD LVCMOS33 } [get_ports { JB[3] }]; #IO_L13N_T2_MRCC_15 Sch=jb[3]
#set_property -dict { PACKAGE_PIN H14   IOSTANDARD LVCMOS33 } [get_ports { JB[4] }]; #IO_L15P_T2_DQS_15 Sch=jb[4]
#set_property -dict { PACKAGE_PIN E16   IOSTANDARD LVCMOS33 } [get_ports { JB[7] }]; #IO_L11N_T1_SRCC_15 Sch=jb[7]
#set_property -dict { PACKAGE_PIN F13   IOSTANDARD LVCMOS33 } [get_ports { JB[8] }]; #IO_L5P_T0_AD9P_15 Sch=jb[8]
#set_property -dict { PACKAGE_PIN G13   IOSTANDARD LVCMOS33 } [get_ports { JB[9] }]; #IO_0_15 Sch=jb[9]
#set_property -dict { PACKAGE_PIN H16   IOSTANDARD LVCMOS33 } [get_ports { JB[10] }]; #IO_L13P_T2_MRCC_15 Sch=jb[10]

##Pmod Header JC
#set_property -dict { PACKAGE_PIN K1    IOSTANDARD LVCMOS33 } [get_ports { JC[1] }]; #IO_L23N_T3_35 Sch=jc[1]
#set_property -dict { PACKAGE_PIN F6    IOSTANDARD LVCMOS33 } [get_ports { JC[2] }]; #IO_L19N_T3_VREF_35 Sch=jc[2]
#set_property -dict { PACKAGE_PIN J2    IOSTANDARD LVCMOS33 } [get_ports { JC[3] }]; #IO_L22N_T3_35 Sch=jc[3]
#set_property -dict { PACKAGE_PIN G6    IOSTANDARD LVCMOS33 } [get_ports { JC[4] }]; #IO_L19P_T3_35 Sch=jc[4]
#set_property -dict { PACKAGE_PIN E7    IOSTANDARD LVCMOS33 } [get_ports { JC[7] }]; #IO_L6P_T0_35 Sch=jc[7]
#set_property -dict { PACKAGE_PIN J3    IOSTANDARD LVCMOS33 } [get_ports { JC[8] }]; #IO_L22P_T3_35 Sch=jc[8]
#set_property -dict { PACKAGE_PIN J4    IOSTANDARD LVCMOS33 } [get_ports { JC[9] }]; #IO_L21P_T3_DQS_35 Sch=jc[9]
#set_property -dict { PACKAGE_PIN E6    IOSTANDARD LVCMOS33 } [get_ports { JC[10] }]; #IO_L5P_T0_AD13P_35 Sch=jc[10]

##Pmod Header JD
#set_property -dict { PACKAGE_PIN H4    IOSTANDARD LVCMOS33 } [get_ports { gpio_o[0] }]; #IO_L21N_T3_DQS_35 Sch=jd[1]
#set_property -dict { PACKAGE_PIN H1    IOSTANDARD LVCMOS33 } [get_ports { JD[2] }]; #IO_L17P_T2_35 Sch=jd[2]
#set_property -dict { PACKAGE_PIN G1    IOSTANDARD LVCMOS33 } [get_ports { JD[3] }]; #IO_L17N_T2_35 Sch=jd[3]
#set_property -dict { PACKAGE_PIN G3    IOSTANDARD LVCMOS33 } [get_ports { JD[4] }]; #IO_L20N_T3_35 Sch=jd[4]
#set_property -dict { PACKAGE_PIN H2    IOSTANDARD LVCMOS33 } [get_ports { JD[7] }]; #IO_L15P_T2_DQS_35 Sch=jd[7]
#set_property -dict { PACKAGE_PIN G4    IOSTANDARD LVCMOS33 } [get_ports { JD[8] }]; #IO_L20P_T3_35 Sch=jd[8]
#set_property -dict { PACKAGE_PIN G2    IOSTANDARD LVCMOS33 } [get_ports { JD[9] }]; #IO_L15N_T2_DQS_35 Sch=jd[9]
#set_property -dict { PACKAGE_PIN F3    IOSTANDARD LVCMOS33 } [get_ports { JD[10] }]; #IO_L13N_T2_MRCC_35 Sch=jd[10]

##Pmod Header JXADC
#set_property -dict { PACKAGE_PIN A14   IOSTANDARD LVCMOS33 } [get_ports { XA_N[1] }]; #IO_L9N_T1_DQS_AD3N_15 Sch=xa_n[1]
#set_property -dict { PACKAGE_PIN A13   IOSTANDARD LVCMOS33 } [get_ports { XA_P[1] }]; #IO_L9P_T1_DQS_AD3P_15 Sch=xa_p[1]
#set_property -dict { PACKAGE_PIN A16   IOSTANDARD LVCMOS33 } [get_ports { XA_N[2] }]; #IO_L8N_T1_AD10N_15 Sch=xa_n[2]
#set_property -dict { PACKAGE_PIN A15   IOSTANDARD LVCMOS33 } [get_ports { XA_P[2] }]; #IO_L8P_T1_AD10P_15 Sch=xa_p[2]
#set_property -dict { PACKAGE_PIN B17   IOSTANDARD LVCMOS33 } [get_ports { XA_N[3] }]; #IO_L7N_T1_AD2N_15 Sch=xa_n[3]
#set_property -dict { PACKAGE_PIN B16   IOSTANDARD LVCMOS33 } [get_ports { XA_P[3] }]; #IO_L7P_T1_AD2P_15 Sch=xa_p[3]
#set_property -dict { PACKAGE_PIN A18   IOSTANDARD LVCMOS33 } [get_ports { XA_N[4] }]; #IO_L10N_T1_AD11N_15 Sch=xa_n[4]
#set_property -dict { PACKAGE_PIN B18   IOSTANDARD LVCMOS33 } [get_ports { XA_P[4] }]; #IO_L10P_T1_AD11P_15 Sch=xa_p[4]

##VGA Connector
#set_property -dict { PACKAGE_PIN A3    IOSTANDARD LVCMOS33 } [get_ports { VGA_R[0] }]; #IO_L8N_T1_AD14N_35 Sch=vga_r[0]
#set_property -dict { PACKAGE_PIN B4    IOSTANDARD LVCMOS33 } [get_ports { VGA_R[1] }]; #IO_L7N_T1_AD6N_35 Sch=vga_r[1]
#set_property -dict { PACKAGE_PIN C5    IOSTANDARD LVCMOS33 } [get_ports { VGA_R[2] }]; #IO_L1N_T0_AD4N_35 Sch=vga_r[2]
#set_property -dict { PACKAGE_PIN A4    IOSTANDARD LVCMOS33 } [get_ports { VGA_R[3] }]; #IO_L8P_T1_AD14P_35 Sch=vga_r[3]
#set_property -dict { PACKAGE_PIN C6    IOSTANDARD LVCMOS33 } [get_ports { VGA_G[0] }]; #IO_L1P_T0_AD4P_35 Sch=vga_g[0]
#set_property -dict { PACKAGE_PIN A5    IOSTANDARD LVCMOS33 } [get_ports { VGA_G[1] }]; #IO_L3N_T0_DQS_AD5N_35 Sch=vga_g[1]
#set_property -dict { PACKAGE_PIN B6    IOSTANDARD LVCMOS33 } [get_ports { VGA_G[2] }]; #IO_L2N_T0_AD12N_35 Sch=vga_g[2]
#set_property -dict { PACKAGE_PIN A6    IOSTANDARD LVCMOS33 } [get_ports { VGA_G[3] }]; #IO_L3P_T0_DQS_AD5P_35 Sch=vga_g[3]
#set_property -dict { PACKAGE_PIN B7    IOSTANDARD LVCMOS33 } [get_ports { VGA_B[0] }]; #IO_L2P_T0_AD12P_35 Sch=vga_b[0]
#set_property -dict { PACKAGE_PIN C7    IOSTANDARD LVCMOS33 } [get_ports { VGA_B[1] }]; #IO_L4N_T0_35 Sch=vga_b[1]
#set_property -dict { PACKAGE_PIN D7    IOSTANDARD LVCMOS33 } [get_ports { VGA_B[2] }]; #IO_L6N_T0_VREF_35 Sch=vga_b[2]
#set_property -dict { PACKAGE_PIN D8    IOSTANDARD LVCMOS33 } [get_ports { VGA_B[3] }]; #IO_L4P_T0_35 Sch=vga_b[3]
#set_property -dict { PACKAGE_PIN B11   IOSTANDARD LVCMOS33 } [get_ports { VGA_HS }]; #IO_L4P_T0_15 Sch=vga_hs
#set_property -dict { PACKAGE_PIN B12   IOSTANDARD LVCMOS33 } [get_ports { VGA_VS }]; #IO_L3N_T0_DQS_AD1N_15 Sch=vga_vs

##Micro SD Connector
#set_property -dict { PACKAGE_PIN E2    IOSTANDARD LVCMOS33 } [get_ports { SD_RESET }]; #IO_L14P_T2_SRCC_35 Sch=sd_reset
#set_property -dict { PACKAGE_PIN A1    IOSTANDARD LVCMOS33 } [get_ports { SD_CD }]; #IO_L9N_T1_DQS_AD7N_35 Sch=sd_cd
#set_property -dict { PACKAGE_PIN B1    IOSTANDARD LVCMOS33 } [get_ports { SD_SCK }]; #IO_L9P_T1_DQS_AD7P_35 Sch=sd_sck
#set_property -dict { PACKAGE_PIN C1    IOSTANDARD LVCMOS33 } [get_ports { SD_CMD }]; #IO_L16N_T2_35 Sch=sd_cmd
#set_property -dict { PACKAGE_PIN C2    IOSTANDARD LVCMOS33 } [get_ports { SD_DAT[0] }]; #IO_L16P_T2_35 Sch=sd_dat[0]
#set_property -dict { PACKAGE_PIN E1    IOSTANDARD LVCMOS33 } [get_ports { SD_DAT[1] }]; #IO_L18N_T2_35 Sch=sd_dat[1]
#set_property -dict { PACKAGE_PIN F1    IOSTANDARD LVCMOS33 } [get_ports { SD_DAT[2] }]; #IO_L18P_T2_35 Sch=sd_dat[2]
#set_property -dict { PACKAGE_PIN D2    IOSTANDARD LVCMOS33 } [get_ports { SD_DAT[3] }]; #IO_L14N_T2_SRCC_35 Sch=sd_dat[3]

##Accelerometer
#set_property -dict { PACKAGE_PIN E15   IOSTANDARD LVCMOS33 } [get_ports { ACL_MISO }]; #IO_L11P_T1_SRCC_15 Sch=acl_miso
#set_property -dict { PACKAGE_PIN F14   IOSTANDARD LVCMOS33 } [get_ports { ACL_MOSI }]; #IO_L5N_T0_AD9N_15 Sch=acl_mosi
#set_property -dict { PACKAGE_PIN F15   IOSTANDARD LVCMOS33 } [get_ports { ACL_SCLK }]; #IO_L14P_T2_SRCC_15 Sch=acl_sclk
#set_property -dict { PACKAGE_PIN D15   IOSTANDARD LVCMOS33 } [get_ports { ACL_CSN }]; #IO_L12P_T1_MRCC_15 Sch=acl_csn
#set_property -dict { PACKAGE_PIN B13   IOSTANDARD LVCMOS33 } [get_ports { ACL_INT[1] }]; #IO_L2P_T0_AD8P_15 Sch=acl_int[1]
#set_property -dict { PACKAGE_PIN C16   IOSTANDARD LVCMOS33 } [get_ports { ACL_INT[2] }]; #IO_L20P_T3_A20_15 Sch=acl_int[2]

##Temperature Sensor
#set_property -dict { PACKAGE_PIN C14   IOSTANDARD LVCMOS33 } [get_ports { TMP_SCL }]; #IO_L1N_T0_AD0N_15 Sch=tmp_scl
#set_property -dict { PACKAGE_PIN C15   IOSTANDARD LVCMOS33 } [get_ports { TMP_SDA }]; #IO_L12N_T1_MRCC_15 Sch=tmp_sda
#set_property -dict { PACKAGE_PIN D13   IOSTANDARD LVCMOS33 } [get_ports { TMP_INT }]; #IO_L6N_T0_VREF_15 Sch=tmp_int
#set_property -dict { PACKAGE_PIN B14   IOSTANDARD LVCMOS33 } [get_ports { TMP_CT }]; #IO_L2N_T0_AD8N_15 Sch=tmp_ct

##Omnidirectional Microphone
#set_property -dict { PACKAGE_PIN J5    IOSTANDARD LVCMOS33 } [get_ports { M_CLK }]; #IO_25_35 Sch=m_clk
#set_property -dict { PACKAGE_PIN H5    IOSTANDARD LVCMOS33 } [get_ports { M_DATA }]; #IO_L24N_T3_35 Sch=m_data
#set_property -dict { PACKAGE_PIN F5    IOSTANDARD LVCMOS33 } [get_ports { M_LRSEL }]; #IO_0_35 Sch=m_lrsel

##PWM Audio Amplifier
#set_property -dict { PACKAGE_PIN A11   IOSTANDARD LVCMOS33 } [get_ports { AUD_PWM }]; #IO_L4N_T0_15 Sch=aud_pwm
#set_property -dict { PACKAGE_PIN D12   IOSTANDARD LVCMOS33 } [get_ports { AUD_SD }]; #IO_L6P_T0_15 Sch=aud_sd

##USB-RS232 Interface
#set_property -dict { PACKAGE_PIN C4    IOSTANDARD LVCMOS33 } [get_ports { UART_TXD_IN }]; #IO_L7P_T1_AD6P_35 Sch=uart_txd_in
set_property -dict { PACKAGE_PIN D4    IOSTANDARD LVCMOS33 } [get_ports { uart_tx_o }]; #IO_L11N_T1_SRCC_35 Sch=uart_rxd_out
#set_property -dict { PACKAGE_PIN D3    IOSTANDARD LVCMOS33 } [get_ports { UART_CTS }]; #IO_L12N_T1_MRCC_35 Sch=uart_cts
#set_property -dict { PACKAGE_PIN E5    IOSTANDARD LVCMOS33 } [get_ports { UART_RTS }]; #IO_L5N_T0_AD13N_35 Sch=uart_rts

##USB HID (PS/2)
#set_property -dict { PACKAGE_PIN F4    IOSTANDARD LVCMOS33 } [get_ports { PS2_CLK }]; #IO_L13P_T2_MRCC_35 Sch=ps2_clk
#set_property -dict { PACKAGE_PIN B2    IOSTANDARD LVCMOS33 } [get_ports { PS2_DATA }]; #IO_L10N_T1_AD15N_35 Sch=ps2_data

##SMSC Ethernet PHY
#set_property -dict { PACKAGE_PIN C9    IOSTANDARD LVCMOS33 } [get_ports { ETH_MDC }]; #IO_L11P_T1_SRCC_16 Sch=eth_mdc
#set_property -dict { PACKAGE_PIN A9    IOSTANDARD LVCMOS33 } [get_ports { ETH_MDIO }]; #IO_L14N_T2_SRCC_16 Sch=eth_mdio
#set_property -dict { PACKAGE_PIN B3    IOSTANDARD LVCMOS33 } [get_ports { ETH_RSTN }]; #IO_L10P_T1_AD15P_35 Sch=eth_rstn
#set_property -dict { PACKAGE_PIN D9    IOSTANDARD LVCMOS33 } [get_ports { ETH_CRSDV }]; #IO_L6N_T0_VREF_16 Sch=eth_crsdv
#set_property -dict { PACKAGE_PIN C10   IOSTANDARD LVCMOS33 } [get_ports { ETH_RXERR }]; #IO_L13N_T2_MRCC_16 Sch=eth_rxerr
#set_property -dict { PACKAGE_PIN C11   IOSTANDARD LVCMOS33 } [get_ports { ETH_RXD[0] }]; #IO_L13P_T2_MRCC_16 Sch=eth_rxd[0]
#set_property -dict { PACKAGE_PIN D10   IOSTANDARD LVCMOS33 } [get_ports { ETH_RXD[1] }]; #IO_L19N_T3_VREF_16 Sch=eth_rxd[1]
#set_property -dict { PACKAGE_PIN B9    IOSTANDARD LVCMOS33 } [get_ports { ETH_TXEN }]; #IO_L11N_T1_SRCC_16 Sch=eth_txen
#set_property -dict { PACKAGE_PIN A10   IOSTANDARD LVCMOS33 } [get_ports { ETH_TXD[0] }]; #IO_L14P_T2_SRCC_16 Sch=eth_txd[0]
#set_property -dict { PACKAGE_PIN A8    IOSTANDARD LVCMOS33 } [get_ports { ETH_TXD[1] }]; #IO_L12N_T1_MRCC_16 Sch=eth_txd[1]
#set_property -dict { PACKAGE_PIN D5    IOSTANDARD LVCMOS33 } [get_ports { ETH_REFCLK }]; #IO_L11P_T1_SRCC_35 Sch=eth_refclk
#set_property -dict { PACKAGE_PIN B8    IOSTANDARD LVCMOS33 } [get_ports { ETH_INTN }]; #IO_L12P_T1_MRCC_16 Sch=eth_intn

##Quad SPI Flash
#set_property -dict { PACKAGE_PIN K17   IOSTANDARD LVCMOS33 } [get_ports { QSPI_DQ[0] }]; #IO_L1P_T0_D00_MOSI_14 Sch=qspi_dq[0]
#set_property -dict { PACKAGE_PIN K18   IOSTANDARD LVCMOS33 } [get_ports { QSPI_DQ[1] }]; #IO_L1N_T0_D01_DIN_14 Sch=qspi_dq[1]
#set_property -dict { PACKAGE_PIN L14   IOSTANDARD LVCMOS33 } [get_ports { QSPI_DQ[2] }]; #IO_L2P_T0_D02_14 Sch=qspi_dq[2]
#set_property -dict { PACKAGE_PIN M14   IOSTANDARD LVCMOS33 } [get_ports { QSPI_DQ[3] }]; #IO_L2N_T0_D03_14 Sch=qspi_dq[3]
#set_property -dict { PACKAGE_PIN L13   IOSTANDARD LVCMOS33 } [get_ports { QSPI_CSN }]; #IO_L6P_T0_FCS_B_14 Sch=qspi_csn
```

</details>

<p></p>

Constaint dosyamızı yükledikten sonra Nexys A7-100T kartımızı seçmeliyiz.

<p align="center">
  <img src="/board-selection.png"/>
</p>

<p align="center">
   Vivado kart seçimi
</p>

Projemizi oluşturduk. Şimdi saat frekansını düşürmek için Clocking Wizard IP eklemeliyiz. Flow Navigator menüsü altında bulunan IP Catalog'a tıklıyoruz. Açılan menüden Clocking Wizard IP'sine çift tıklıyoruz.

<p align="center">
  <img src="/ip-catalog.png"/>
</p>

<p align="center">
   IP Catalog menüsü
</p>

Açılan pencereden IP'ye <code>clk_50mhz</code> adını veriyoruz, ardından output sekmesi altında yer alan Requested Output Freq metin kutusuna 50.000 değerini giriyoruz.

<p align="center">
  <img src="/clocking-wizard.png"/>
</p>

<p align="center">
   Clocking Wizard IP penceresi
</p>

Bu işlemin ardından generate butonuna tıklamalıyız.

<p align="center">
  <img src="/generate-outputs.png"/>
</p>

<p align="center">
   Generate Output Products penceresi
</p>

Şimdi sentez, implementation ve generate bitstream komutlarını vererek matrak işlemcimizi FPGA üzerinde çalıştırmaya hazırız.

## Kara Şimşek Uygulaması

LED'leri kullanarak birçok uygulama yapabiliriz. Biz klasik Kara Şimşek projesini tercih edeceğiz. Bunun için aşağıdaki kodu hazırladık.

knight.c:

```c
#include "matrak.h"

#define HIGH 1
#define LOW 0

void main(void) {

   while (1) {
      for (int i = 0; i < 8; i++) {
         gpio_write(i, HIGH);
         delay_ms(100);
         gpio_write(i, LOW);
      }
      for (int j = 7; j > -1; j--) {
         gpio_write(j, HIGH);
         delay_ms(100);
         gpio_write(j, LOW);
      }
   }
}
```

Makefile dosyamızı düzenledikten sonra <code>make knight.hex</code> komutunu vererek kodumuzu derliyoruz. knight.hex dosyasının içeriğini program.mem dosyasına aktarıp Vivado üzerinden generate bitstream'e tıklıyoruz. Ardından Hardware Manager penceresinden üretilen bitstream dosyasını FPGA'ya yüklüyoruz.

<details>
<summary>knight.hex: <mark>kodu göstermek için tıklayın</mark></summary>

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
00100593
fec42503
e45ff0ef
06400513
eb9ff0ef
00000593
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
00100593
fe842503
e01ff0ef
06400513
e75ff0ef
00000593
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

Ve sonuç:

<p align="center">
  <img src="/nexys-knight-rider.gif"/>
</p>

<p align="center">
   Kara Şimşek, basit ve etkileyici. Gerçek bir klasik.
</p>

<p align="center">
  <img src="/knightrider.jpg" width="460" height="624"/>
</p>

## UART Uygulaması

Elimiz değmişken UART çevrebirimimizi de test edelim. Takip edeceğimiz işlemler önceki uygulama ile aynı. UART test kodu aşağıda verilmiştir.

uart.c:

```c
#include "matrak.h"

void main(void) {

   put_str("Iskeleden uzaklasan bir gemi\n");
   put_str("Hatirlatir bana mazide kalan gunlerimi\n");
   put_str("Gordugum su mavi deniz ufkumu aydinlatir\n");
   put_str("Ucup giden bir marti yitirdiklerimi\n");

   while(1);
}
```

<details>
<summary>uart.hex: <mark>kodu göstermek için tıklayın</mark></summary>

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
ff010113
00112623
00812423
01010413
26800513
f8dff0ef
28800513
f85ff0ef
2b000513
f7dff0ef
2dc00513
f75ff0ef
0000006f
656b7349
6564656c
7a75206e
616c6b61
206e6173
20726962
696d6567
0000000a
69746148
74616c72
62207269
20616e61
697a616d
6b206564
6e616c61
6e756720
6972656c
000a696d
64726f47
6d756775
20757320
6976616d
6e656420
75207a69
6d756b66
79612075
6c6e6964
72697461
0000000a
70756355
64696720
62206e65
6d207269
69747261
74697920
69647269
72656c6b
0a696d69
00000000
```

</details>

<p></p>

UART çalışıyor gibi görünüyor.

<p align="center">
  <img src="/serial-output-crop.png"/>
</p>

<p></p>

<p align="center">
  <img src="/matrak-logo.png" width="400" height="212"/>
</p>

<p></p>

Daha yapılacak çok iş var ama şimdilik bitti.

> :scroll: Bu bölümün kodlarına erişmek için [tıklayın](https://github.com/necaticakaci/matrak/tree/main/episodes/ep5/nexysa7).

---

## Kaynaklar

* [RISC-V Unprivileged Specification](https://github.com/riscv/riscv-isa-manual/releases/tag/Ratified-IMAFDQC)
* [Digital Design and Computer Architecture: RISC-V Edition](https://pages.hmc.edu/harris/ddca/)
* [RISC-V Kopya Kağıdı](https://github.com/jameslzhu/riscv-card/blob/master/riscv-card.pdf)
* [Bilgisayar Mimarsi Dersleri, Oğuz Ergin](https://www.youtube.com/playlist?list=PLvNq8wrSYGAU6CF4UleG6HbXa9paQDsLK)
* [VHDL ve FPGA Dersleri, Mehmet Burak Aykenar](https://github.com/mbaykenar/apis_anatolia/)
* [learn-fpga, Bruno Levy](https://github.com/BrunoLevy/learn-fpga/)
* [mystic_riscv64, Muhammed Kocaoğlu](https://github.com/muhammedkocaoglu/mystic_riscv64/)
* [riscv-simple-sv](https://github.com/tilk/riscv-simple-sv)
