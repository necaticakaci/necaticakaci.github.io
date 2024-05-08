---
title: "RISC-V İşlemci Tasarımı - Bölüm 7: Açık Kaynaklı FPGA Akışı"
date: 2024-05-07
description: Bu bölümde Matrak işlemcimizi, açık kaynaklı FPGA araçlarını kullanarak Gowin FPGA üzerinde çalıştırıyoruz.
math: true
---

Bu bölüme kadar FPGA akışında Xilinx Vivado yazılımını kullanmıştık. Fakat Xilinx FPGA kartlarının maliyeti, öğrencilerin ve hobicilerin bu iş için ayıracağı bütçeyi aşabilir. Bu noktada FPGA çalışmalarına giriş yapabileceğimiz uygun maliyetli kartlara ihtiyaç duyuyoruz. Yakın döneme kadar birkaç üreticinin tekelinde bulunan FPGA pazarında, son dönemde bir çok küçük çaplı üreticileri görmeye başladık. Bu sayede büyük üreticilerin neredeyse CPLD kartlarının fiyatlarında giriş seviyesi FPGA kartları piyasaya sürülmüş oldu. Gowin de bu girişimlerden biri olan Çin menşeili bir FPGA yongası üreticisi. Gowin FPGA'lara Sipeed Tang Nano serisi kartlarda rastlıyoruz. Tang Nano FPGA kartları nispeten uygun maliyetli ve -bu yazı yazıldığı esnada- ülkemiz üzerinden de temin edilebiliyor. Ayrıca Tang Nano kartları açık kaynaklı araçları destekliyor. Tabii ki Gowin FPGA'ların da sahipli (ücretsiz lisanslı) IDE'leri var, ancak biz bu yazıda açık kaynaklı araçlar üzerinde duracağız.

:pushpin: Açık kaynaklı FPGA araçlarının üretici tarafından resmi olarak desteklenmediğini belirtelim. Bu durum bazı uyumsuzluklar ve sorunlar yaşayabileceğimiz anlamına gelebilir.

Tang Nano FPGA geliştirme kartlarının 1K, 4K, 9K ve 20K gibi bir çok versiyonu var. Ben Matrak işlemcimizi çalıştırmak için Tang Nano 9K kartını temin ettim. 9K ibaresi kartta yer alan FPGA'nın 8640 LUT4 (Look Up Table) içerdiğine atıf yapıyor. Eğer elinizde 20K versiyonu mevcutsa muhtemelen küçük değişikliklerle bu projeyi çalıştırabilirsiniz.

<p align="center">
  <img src="/tang-nano-9k.png"/>
</p>

<p align="center">
  Sipeed Tang Nano 9K
</p>

Tang Nano 9K kartının merkezinde Gowin GW1NR-9 FPGA yongası bulunuyor. GW1NR-9 yongası, 8640 LUT4'ün yanı sıra 6480 flip-flop, çeşitli boyutlarda SRAM yapıları ve dahili flash içeriyor. Ayrıca bu yonga System in Package (SiP) yapısında bir SDRAM de içeriyor.

Duyduklarımız güzel. Bir FPGA için önemli bileşenler tek yongada toplanmış. Geliştirme kartında bulunan diğer parçalar şunlar:

* HDMI konnektörü,
* USB-C konnektörü
* LCD konnektörleri
* 2 adet buton
* 6 adet LED
* USB JTAG-UART yongası
* 32 Mbit SPI flash
* 27 Mhz kristal osilatör

FPGA'yı harici programlayıcıya ihtiyaç duymadan USB-C üzerinden programlayabiliyoruz. FPGA'yı programlamak için kullandığımız bitstream dosyasını dahili veya harici flash belleğe yazabiliriz.

<p align="center">
  <img src="/board-compare.jpg"/>
</p>

<p align="center">
  Soldan sağa; Arduino Nano, Deneyap Mini, Raspberry Pi Pico, Tang Nano 9K
</p>

Gowin FPGA'ları kullanmak için açık kaynaklı sentez ve place and route araçlarını bilgisyarımıza kuracağız. Kurulum işlemi, tıpkı [5. bölümde](/posts/riscv-5) değindiğimiz RISC-V Toolchain paketinin kurulumu gibi derleme gerektiriyor. Eğer uğraşmak istemezseniz derlenmiş araçların bulunduğu [oss-cad-suite](https://github.com/YosysHQ/oss-cad-suite-build) paketini indirebilirsiniz. Paketi arşivden çıkarttıktan sonra dizini (<code>kurulum_dizini/oss-cad-suite/bin</code>) Path'e eklemeyi unutmayın.

Eğer sadece ihtiyacımız olanları derleyerek kurmayı tercih edersek; [Yosys](https://github.com/yosyshq/yosys), [nextpnr-gowin](https://github.com/YosysHQ/nextpnr#nextpnr-himbaechel), [OpenFPGALoader](https://github.com/trabucayre/openFPGALoader) araçlarına ihtiyacımız olacak. Ben daha pratik olacağı için önceden derlenmiş paketi tercih ettim.

Son olarak Apicula projesini kuracağız. Kurulum için uçbirime <code>pip install apycula</code> komutunu giriyoruz.

Yosys: Sentez aracı

nextpnr-gowin: Gowin FPGA için Place ve Route araçları

Apicula: Gowin FPGA bitstream formatı için araçlar

OpenFPGALoader: FPGA programlama aracı

> :pushpin: OpenFPGALoader aracının FPGA kartına erişebilmesi için aşağıdaki adımları uygulamış olduğunuzdan emin olun.

[99-openfpgaloader.rules](https://github.com/trabucayre/openFPGALoader/blob/master/99-openfpgaloader.rules) dosyasını indirin ve dosyanın olduğu dizinde uçbirimi açıp aşağıdaki komutları girin.

```
$ sudo cp 99-openfpgaloader.rules /etc/udev/rules.d/
$ sudo udevadm control --reload-rules && sudo udevadm trigger # force udev to take new rule
$ sudo usermod -a $USER -G plugdev # add user to plugdev group
```

## İşlemcinin Düzenlenmesi

Matrak işlemcimizi Tang Nano FPGA kartında çalıştırabilmek için donanımda bazı değişiklikler yapacağız. Bunlardan birincisi, sentezden geçebilmek için bellek boyutunu azaltmak olacak. Belleğimizin kapasitesini 640 bayt olacak şekilde güncelliyoruz.

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

   // 160x32 bit = 5120 bit = 640 bayt
   reg [31:0] mem [159:0];

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

Tang Nano kartı üzerinde 27 MHz saat frekansında çalışan bir osilatör bulunuyor. İşlemcimiz bu saat frekansında sentezden geçemediği için PLL yardımıyla saat frekansını düşürmemiz gerekiyor. Neyse ki açık kaynak araçlarla PLL IP'si üretebiliyoruz. Frekans tam olarak bölünebildiği için saati 18 MHz'e düşüren bir PLL tercih ettik. PLL modülünü oluşturmak için uçbirime <code>gowin_pll -i 27 -o 18</code> komutunu girebilirsiniz.

> :warning: gowin_pll aracından ürettirilen PLL modülü bazen garip bir şekilde placement aşamasında sorunlara neden olabiliyor.

pll.v:

```verilog
/**
* PLL configuration
*
* This Verilog module was generated automatically
* using the gowin-pll tool.
* Use at your own risk.
*
* Target-Device:                GW1NR-9 C6/I5
* Given input frequency:        27.000 MHz
* Requested output frequency:   18.000 MHz
* Achieved output frequency:    18.000 MHz
*/

module pll(
   input  clock_in,
   output clock_out,
   output locked
);

   rPLL #(
      .FCLKIN("27.0"),
      .IDIV_SEL(2), // -> PFD = 9.0 MHz (range: 3-400 MHz)
      .FBDIV_SEL(1), // -> CLKOUT = 18.0 MHz (range: 400-600 MHz)
      .ODIV_SEL(32) // -> VCO = 576.0 MHz (range: 600-1200 MHz)
   ) pll (.CLKOUTP(), .CLKOUTD(), .CLKOUTD3(), .RESET(1'b0), .RESET_P(1'b0), .CLKFB(1'b0), .FBDSEL(6'b0), .IDSEL(6'b0), .ODSEL(6'b0), .PSDA(4'b0), .DUTYDA(4'b0), .FDLY(4'b0),
      .CLKIN(clock_in), // 27.0 MHz
      .CLKOUT(clock_out), // 18.0 MHz
      .LOCK(locked)
   );

endmodule
```

Tang Nano kartı üzerinde S1 ve S2 olarak isimlendirilen iki adet buton bulunuyor. İşlemcimizin reset sinyalini S1 butonuna bağlayacağız. Bu butona basıldığında ilgili pin sıfıra çekiliyor, bu sebepten ötürü daha önceden "active high" olarak tasarladığımız reset sinyalini terse çevirerek "active low" haline getiriyoruz.

<p align="center">
  <img src="/tang-nano-buttons.png"/>
</p>

<p align="center">
  Tang Nano 9K buton bağlantı şeması
</p>

Saat ve reset sinyallerimizdeki değişikliklerin ardından üst modülümüz aşağıdaki gibi güncelleniyor.

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

   // Reset tersleniyor
   wire rst_n = ~rst_i;

   // PLL bağlantıları
   wire clk;

   pll p1 (
      .clock_in(clk_i),
      .clock_out(clk),
      .locked()
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
      .clk_i(clk),
      .rst_i(rst_n),
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
      .clk_i(clk),
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
      .clk_i(clk),
      .rst_i(rst_n),
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
      .clk_i(clk),
      .rst_i(rst_n),
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
      .clk_i(clk),
      .rst_i(rst_n),
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

Hatırlayacak olursak işlemcimize çevrebirimi olarak bağlı 8 adet çıkış pinimiz mevcuttu. Bu pinlerin 6'sını kartta yer alan LED'lere bağlayacağız, kalan 2 pini ise kartın boştaki herhangi bir pinine bağlayabiliriz. LED'leri yakmak için ilgili pini sıfıra çekmek gerekiyor. Bu yüzden çıkış birimi modülümüz sıfırlandığında artık pinlere 0 yerine 1 göndermeli.

<p align="center">
  <img src="/tang-nano-leds.png"/>
</p>

<p align="center">
  Tang Nano 9K LED bağlantı şeması
</p>

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

Diğer modüllerde herhangi bir değişiklik yapmıyoruz.

Donanımda yapılması gereken değişiklikleri halletik. Üst modül sinyallerimiz ile FPGA pinlerini eşleyecek "Constraint" dosyamızı aşağıdaki gibi oluşturabiliriz.

tangnano9k.cst:

```
// 27 Mhz oscillator
IO_LOC "clk_i" 52;

// Onboard button
IO_LOC "rst_i" 3;

// Onboard LEDS
IO_LOC "gpio_o[0]" 10;
IO_LOC "gpio_o[1]" 11;
IO_LOC "gpio_o[2]" 13;
IO_LOC "gpio_o[3]" 14;
IO_LOC "gpio_o[4]" 15;
IO_LOC "gpio_o[5]" 16;

IO_LOC "gpio_o[6]" 25;
IO_LOC "gpio_o[7]" 26;

// BL702 chip
IO_LOC "uart_tx_o" 17;
IO_PORT "uart_tx_o" IO_TYPE = LVCMOS33;

// IO_LOC "uart_rx_o" 18;
// IO_PORT "uart_rx_o" IO_TYPE = LVCMOS33;
```

## FPGA Akışı

Verilog kodlarımız hazır olduğuna göre aşağıdaki komutla sentezleme işlemini başlatabiliriz.

```
$ yosys -p "read_verilog matrak.v memory.v clock_counter.v gpio.v uart.v pll.v top.v; synth_gowin -top top -json top.json"
```

Ardından placement ve routing işlemlerini gerçekleştiriyoruz. <code>-\-freq 18</code> ifadesini PLL çıkışından elde edilen 18 MHz'lik saat frekansımızı belirtmek için ekledik.

```
$ nextpnr-gowin --json top.json --freq 18 --write top_pnr.json --device GW1NR-LV9QN88PC6/I5 --family GW1N-9C --cst tangnano9k.cst
```

Son olarak bitstream oluşturuyoruz.

```
$ gowin_pack -d GW1N-9C -o top.fs top_pnr.json
```

Şimdi tasarımızı FPGA kartına yüklemeye hazırız. Aşağıdaki komutu kullanarak bitstream dosyasını FPGA'nın SRAM hücrelerine yükleyebiliriz. Bu şekilde yüklediğimizde FPGA'nın gücü kesilirse konfigürasyon uçacaktır.

```
$ openFPGALoader -b tangnano9k top.fs
```

Dahili flash belleği programlamak istersek aşağıdaki komutu girmeliyiz.

```
$ openFPGALoader -b tangnano9k top.fs -f
```

Harici flash belleği programlamak için de aşağıdaki komutu girebiliriz.

```
$ openFPGALoader -b tangnano9k --external-flash top.fs
```

Gerekli komutların bir Makefile içinde toplanmış hali aşağıda verilmiştir. <code>make all</code> komutuyla bitstream üretebilir, <code>make program</code> komutuyla FPGA'yı programlayabiliriz. 

makefile:

```makefile
BOARD = tangnano9k
FAMILY = GW1N-9C
DEVICE = GW1NR-LV9QN88PC6/I5

all: top.fs

top.json: top.v
	yosys -p "read_verilog matrak.v memory.v clock_counter.v gpio.v uart.v pll.v top.v; synth_gowin -top top -json top.json"

top_pnr.json: top.json
	nextpnr-gowin --json top.json --freq 18 --write top_pnr.json --device ${DEVICE} --family ${FAMILY} --cst ${BOARD}.cst

top.fs: top_pnr.json
	gowin_pack -d ${FAMILY} -o top.fs top_pnr.json

program: top.fs
	openFPGALoader -b ${BOARD} top.fs

clean:
	rm -f top.json top_pnr.json top.fs
```

## FPGA Üzerinde Test

Donanımda yaptığımız değişikliklere uyum sağlamak için yazılımda da ufak düzenlemeler yapmalıyız. Bellek boyutumuz değiştiği için crt0.s başlangıç kodumuzda ve linker scriptimizde hafıza alanımızın üst değerini 640 (0x280) olarak güncelliyoruz.

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
   la x2, 0x280

# Main fonksiyonuna atla
jump_main:
   addi a0, x0, 0
   addi a1, x0, 0
   jal x1, main
```

mem.ld:

```
OUTPUT_ARCH( "riscv" )

MEMORY
{
  RAM : ORIGIN = 0x00000000, LENGTH = 0x280
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

Bellek boyutumuzun kısıtlı olması sebebiyle kütüphanemizi de sadece çıkış ve saat sayacı çevrebirimlerini destekleyecek şekilde kırpıyoruz.

```c
// Matrak Çevrebirim Kütüphanesi

#include <stdint.h>

#define GPIO0_BASE (0x80001000U)
#define GPIO0 ((GPIO*) GPIO0_BASE)

#define CYC_C0_BASE (0x80003000U)
#define CYC_C0 ((CYC_C*) CYC_C0_BASE)

#define CLK_FREQ 18000000

typedef struct {
  volatile uint32_t gpio_output;  // 0x80001000 (RW) OUTPUT REGISTER
} GPIO;

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
```

Test için daha önce hazırlamış olduğumuz kara şimşek kodunu kullanacağız. Buradaki kodda farklı olarak, LED'lerin yakılması için ilgili pinlerin sıfıra çekilmesi gerektiğinden HIGH ifadesini 0 olarak tanımlıyoruz.

knight.c:

```c
#include "matrak.h"

#define HIGH 0
#define LOW 1

void main(void) {

   while (1) {
      for (int i = 0; i < 6; i++) {
         gpio_write(i, HIGH);
         delay_ms(100);
         gpio_write(i, LOW);
      }
      for (int j = 5; j > -1; j--) {
         gpio_write(j, HIGH);
         delay_ms(100);
         gpio_write(j, LOW);
      }
   }
}
```

Test kodumuzu derlemek için hazırladığımız Makefile dosyamız ise aşağıdaki gibi görünüyor.

makefile:

```makefile
CC = riscv32-unknown-elf-gcc
OBJCOPY = riscv32-unknown-elf-objcopy
CFLAGS = -march=rv32i -mabi=ilp32 -nostdlib -Wl,-T,mem.ld

all: knight.hex

crt0.o: crt0.s
	$(CC) $(CFLAGS) -c crt0.s -o crt0.o

knight.elf: crt0.o knight.c
	$(CC) $(CFLAGS) crt0.o knight.c -o knight.elf

knight.bin: knight.elf
	$(OBJCOPY) -O binary knight.elf knight.bin

knight.hex: knight.bin
	python3 bin2hex.py -i knight.bin -o knight.hex

clean:
	rm -f crt0.o knight.elf knight.bin knight.hex
```

Şimdi Matrak işlemcimizi FPGA üzerinde çalıştırmaya hazırız. Bunun için test kodumuzu <code>make all</code> komutunu vererek derliyoruz. Bu işlemin ardından üretilen <code>knight.hex</code> dosyasını işlemcimizin bellek dosyası olan <code>program.mem</code> dosyasına aktarıyoruz. Daha sonra FPGA akışı için hazırladığımız makefile dosyasının bulunduğu dizine geçip yine <code>make all</code> komutu ile bitstream dosyasını elde ediyoruz. Bu noktada <code>make program</code> komutu ile FPGA kartını programlayabilir ve işlemcimizi test edebiliriz.

<p align="center">
  <img src="/gowin-knight-rider.gif"/>
</p>

Bellek alanı bizi oldukça sınırladığı için bir sonraki aşamada yapmamız gereken ilk şey, kart üzerinde yer alan SPI Flash'ı ROM olarak kullanmak olmalı.

> :scroll: Bu bölümün kodlarına erişmek için [tıklayın](https://github.com/necaticakaci/matrak/tree/main/episodes/ep7).
