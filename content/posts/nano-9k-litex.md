---
title: "Tang Nano 9K FPGA ile Litex SoC Denemesi"
date: 2024-10-08
description: Sipeed Tang Nano 9K FPGA kartı için Litex aracını kullanarak RISC-V işlemcili bir SoC üretmeyi deniyoruz. 
math: true
---

FPGA ile çalışırken bazı durumlarda tasarımımızın içine işlemci gömmek isteyebiliriz. FPGA içine herhangi bir RTL tasarım gibi gömülebilen ve FPGA'nın lojik kaynaklarını kullanan işlemcilere "soft-core" denir. Soft-core işlemciler, genelde ihtiyaçlara göre özelleştirilebilir yapıda olurlar. FPGA ile yapılan tasarımlarda sıklıkla kullanıldıklarından birçok FPGA üreticisinin kendi soft-core işlemci IP'leri bulunmaktadır. AMD için Microblaze ve Altera için Nios işlemcilerini örnek olarak verebiliriz. Açık kaynak tarafına baktığımızda ise çeşitli mimarilerde çok sayıda işlemci projesinin olduğunu görüyoruz. Fakat bu projelerin büyük kısmı FPGA üreticilerinin sunduğu çevrebirim ve mikromimari özelleştirme kabiliyetlerinden yoksunlar. Litex projesi, barındırdığı çeşitli açık kaynaklı çevrebirim ve işlemci çekirdekleriyle özelleştirilebilir bir SoC oluşturmaya imkan sağlıyor.

Bu yazıda Tang Nano 9K FPGA için Litex ekosisteminden bir SoC üretip çalıştırmaya değineceğiz. Bunu [RISC-V İşlemci Tasarımı - Bölüm 7: Açık Kaynaklı FPGA Akışı](/posts/riscv-7) bölümünde de kullandığımız Yosys/Apicula araçlarıyla gerçekleştireceğiz.

## Litex ile SoC Üretimi

Öncelikle, eğer kurulu değilse [Litex](https://github.com/enjoy-digital/litex) ve [OSS CAD Suite](https://github.com/YosysHQ/oss-cad-suite-build) araçlarının kurulumlarını yapmalıyız.

Daha sonra içinde çalışmak için bir dizin oluşturuyoruz:

```
$ mkdir example_project
$ cd example_project
```

Seçtiğimiz FPGA kartı Litex tarafından desteklendiği için hazır bir betik kullanabileceğiz. Bu Python betiğini <code>-h</code> argümanı ile çağırıp, kullanımını inceleyebiliriz.

```
$ python3 -m litex_boards.targets.sipeed_tang_nano_9k -h
```

Komutun çıktısı aşağıda verilmiştir.

<details>
<summary><mark>Çıktıyı göstermek için tıklayın</mark></summary>

```
usage: {'description': 'LiteX SoC on Tang Nano 9K.'} [-h] [--toolchain {gowin,apicula}] [--build] [--load]
                                                     [--log-filename LOG_FILENAME] [--log-level LOG_LEVEL] [--flash]
                                                     [--sys-clk-freq SYS_CLK_FREQ]
                                                     [--bios-flash-offset BIOS_FLASH_OFFSET] [--with-spi-sdcard]
                                                     [--with-video-terminal] [--prog-kit PROG_KIT]
                                                     [--output-dir OUTPUT_DIR] [--gateware-dir GATEWARE_DIR]
                                                     [--software-dir SOFTWARE_DIR] [--include-dir INCLUDE_DIR]
                                                     [--generated-dir GENERATED_DIR] [--build-backend BUILD_BACKEND]
                                                     [--no-compile] [--no-compile-software] [--no-compile-gateware]
                                                     [--soc-csv SOC_CSV] [--soc-json SOC_JSON] [--soc-svd SOC_SVD]
                                                     [--memory-x MEMORY_X] [--doc] [--bios-lto]
                                                     [--bios-format {integer,float,double}]
                                                     [--bios-console {full,no-history,no-autocomplete,lite,disable}]
                                                     [--bus-standard BUS_STANDARD] [--bus-data-width BUS_DATA_WIDTH]
                                                     [--bus-address-width BUS_ADDRESS_WIDTH] [--bus-timeout BUS_TIMEOUT]
                                                     [--bus-bursting] [--bus-interconnect BUS_INTERCONNECT]
                                                     [--cpu-type CPU_TYPE] [--cpu-variant CPU_VARIANT]
                                                     [--cpu-reset-address CPU_RESET_ADDRESS] [--cpu-cfu CPU_CFU]
                                                     [--no-ctrl] [--integrated-rom-size INTEGRATED_ROM_SIZE]
                                                     [--integrated-rom-init INTEGRATED_ROM_INIT]
                                                     [--integrated-sram-size INTEGRATED_SRAM_SIZE]
                                                     [--integrated-main-ram-size INTEGRATED_MAIN_RAM_SIZE]
                                                     [--csr-data-width CSR_DATA_WIDTH]
                                                     [--csr-address-width CSR_ADDRESS_WIDTH] [--csr-paging CSR_PAGING]
                                                     [--csr-ordering CSR_ORDERING] [--ident IDENT] [--no-ident-version]
                                                     [--no-uart] [--uart-name UART_NAME] [--uart-baudrate UART_BAUDRATE]
                                                     [--uart-fifo-depth UART_FIFO_DEPTH] [--with-uartbone]
                                                     [--with-jtagbone] [--jtagbone-chain JTAGBONE_CHAIN] [--no-timer]
                                                     [--timer-uptime] [--with-watchdog]
                                                     [--watchdog-width WATCHDOG_WIDTH]
                                                     [--watchdog-reset-delay WATCHDOG_RESET_DELAY] [--l2-size L2_SIZE]

options:
  -h, --help
          show this help message and exit

Target options:
  --toolchain {gowin,apicula}
          FPGA toolchain (gowin or apicula). (default: gowin)
  --build
          Build design. (default: False)
  --load  Load bitstream. (default: False)
  --flash
          Flash Bitstream. (default: False)
  --sys-clk-freq SYS_CLK_FREQ
          System clock frequency. (default: 27000000.0)
  --bios-flash-offset BIOS_FLASH_OFFSET
          BIOS offset in SPI Flash. (default: 0x0)
  --with-spi-sdcard
          Enable SPI-mode SDCard support. (default: False)
  --with-video-terminal
          Enable Video Terminal (HDMI). (default: False)
  --prog-kit PROG_KIT
          Programmer select from Gowin/openFPGALoader. (default: openfpgaloader)

Logging options:
  --log-filename LOG_FILENAME
          Logging filename. (default: None)
  --log-level LOG_LEVEL
          Logging level: debug, info (default), warning error or critical. (default: info)

Builder options:
  --output-dir OUTPUT_DIR
          Base Output directory. (default: None)
  --gateware-dir GATEWARE_DIR
          Output directory for Gateware files. (default: None)
  --software-dir SOFTWARE_DIR
          Output directory for Software files. (default: None)
  --include-dir INCLUDE_DIR
          Output directory for Header files. (default: None)
  --generated-dir GENERATED_DIR
          Output directory for Generated files. (default: None)
  --build-backend BUILD_BACKEND
          Select build backend: litex or edalize. (default: litex)
  --no-compile
          Disable Software and Gateware compilation. (default: False)
  --no-compile-software
          Disable Software compilation only. (default: False)
  --no-compile-gateware
          Disable Gateware compilation only. (default: False)
  --soc-csv SOC_CSV, --csr-csv SOC_CSV
          Write SoC mapping to the specified CSV file. (default: None)
  --soc-json SOC_JSON, --csr-json SOC_JSON
          Write SoC mapping to the specified JSON file. (default: None)
  --soc-svd SOC_SVD, --csr-svd SOC_SVD
          Write SoC mapping to the specified SVD file. (default: None)
  --memory-x MEMORY_X
          Write SoC Memory Regions to the specified Memory-X file. (default: None)
  --doc   Generate SoC Documentation. (default: False)

BIOS options:
  --bios-lto
          Enable BIOS LTO (Link Time Optimization) compilation. (default: False)
  --bios-format {integer,float,double}
          Select BIOS printf format. (default: integer)
  --bios-console {full,no-history,no-autocomplete,lite,disable}
          Select BIOS console config. (default: full)

SoC options:
  --bus-standard BUS_STANDARD
          Select bus standard: wishbone, axi-lite, axi. (default: wishbone)
  --bus-data-width BUS_DATA_WIDTH
          Bus data-width. (default: 32)
  --bus-address-width BUS_ADDRESS_WIDTH
          Bus address-width. (default: 32)
  --bus-timeout BUS_TIMEOUT
          Bus timeout in cycles. (default: 1000000)
  --bus-bursting
          Enable burst cycles on the bus if supported. (default: False)
  --bus-interconnect BUS_INTERCONNECT
          Select bus interconnect: shared (default) or crossbar. (default: shared)
  --cpu-type CPU_TYPE
          Select CPU: None, serv, vexiiriscv, cv32e40p, neorv32, cva6, mor1kx, cv32e41p, kianv, fazyrv, cva5, cortex_m1,
          eos_s3, ibex, marocchino, gowin_emcu, openc906, minerva, firev, femtorv, vexriscv, cortex_m3, naxriscv,
          zynqmp, rocket, zynq7000, microwatt, vexriscv_smp, lm32, gowin_ae350, blackparrot, picorv32. (default:
          vexriscv)
  --cpu-variant CPU_VARIANT
          CPU variant. (default: None)
  --cpu-reset-address CPU_RESET_ADDRESS
          CPU reset address (Boot from Integrated ROM by default). (default: None)
  --cpu-cfu CPU_CFU
          Optional CPU CFU file/instance to add to the CPU. (default: None)
  --no-ctrl
          Disable Controller. (default: False)
  --integrated-rom-size INTEGRATED_ROM_SIZE
          Size/Enable the integrated (BIOS) ROM (Automatically resized to BIOS size when smaller). (default: 131072)
  --integrated-rom-init INTEGRATED_ROM_INIT
          Integrated ROM binary initialization file (override the BIOS when specified). (default: None)
  --integrated-sram-size INTEGRATED_SRAM_SIZE
          Size/Enable the integrated SRAM. (default: 8192)
  --integrated-main-ram-size INTEGRATED_MAIN_RAM_SIZE
          size/enable the integrated main RAM. (default: None)
  --csr-data-width CSR_DATA_WIDTH
          CSR bus data-width (8 or 32). (default: 32)
  --csr-address-width CSR_ADDRESS_WIDTH
          CSR bus address-width. (default: 14)
  --csr-paging CSR_PAGING
          CSR bus paging. (default: 2048)
  --csr-ordering CSR_ORDERING
          CSR registers ordering (big or little). (default: big)
  --ident IDENT
          SoC identifier. (default: None)
  --no-ident-version
          Disable date/time in SoC identifier. (default: False)
  --no-uart
          Disable UART. (default: False)
  --uart-name UART_NAME
          UART type/name. (default: serial)
  --uart-baudrate UART_BAUDRATE
          UART baudrate. (default: 115200)
  --uart-fifo-depth UART_FIFO_DEPTH
          UART FIFO depth. (default: 16)
  --with-uartbone
          Enable UARTbone. (default: False)
  --with-jtagbone
          Enable Jtagbone support. (default: False)
  --jtagbone-chain JTAGBONE_CHAIN
          Jtagbone chain index. (default: 1)
  --no-timer
          Disable Timer. (default: False)
  --timer-uptime
          Add an uptime capability to Timer. (default: False)
  --with-watchdog
          Enable Watchdog. (default: False)
  --watchdog-width WATCHDOG_WIDTH
          Watchdog width. (default: 32)
  --watchdog-reset-delay WATCHDOG_RESET_DELAY
          Watchdog width. (default: None)
  --l2-size L2_SIZE
          L2 cache size. (default: 8192)
```
</details>

<p></p>

Açık kaynaklı FPGA araçlarını kullanmak istediğimiz için <code>-\-toolchain apicula</code> argümanını vermeliyiz. Sistemin derlenmesi için <code>-\-build</code> ifadesini ekliyoruz:

```
$ python3 -m litex_boards.targets.sipeed_tang_nano_9k --toolchain apicula --build
```

İlk denememiz ne yazık ki hatayla sonuçlanıyor.:confused:

Çıktıyı incelediğimizde, hatanın sentez aşamasında ortaya çıktığını görüyoruz. Yosys bir sinyal için "Unconstrained IO" hatası veriyor.

Hata çıktısı:

```
5. Executing JSON backend.

Warnings: 2 unique messages, 2 total
End of script. Logfile hash: 2f6167a4b0, CPU: user 24.10s system 0.52s, MEM: 965.94 MB peak
Yosys 0.45+148 (git sha1 1bf908dea, clang++ 14.0.0-1ubuntu1.1 -fPIC -O3)
Time spent: 53% 1x abc9_exe (28 sec), 13% 1x autoname (7 sec), ...
Info: Using uarch 'gowin' for device 'GW1NR-LV9QN88PC6/I5'

Info: Reading constraints...
Info: Create constant nets...
Info: Modify LUTs...
Info: Pack IOBs...
ERROR: Unconstrained IO:O_psram_reset_n_OBUF_O_1
0 warnings, 1 error
```

Hatayı çözmek için, derleme komutu verildiğinde oluşturulan <code>example_project/build/sipeed_tang_nano_9k</code> dizinine göz gezdiriyoruz. Burada <code>gateware</code> ve <code>software</code> adında iki dizin bulunuyor. Sentezlenecek Verilog kodları <code>gateware</code> dizininde yer alıyor. Burada bulunan Yosys betiğinden, sentezlenen dosyaların <code>VexRiscv.v</code> ve <code>sipeed_tang_nano_9k.v</code> olduğunu öğreniyoruz.

sipeed_tang_nano_9k.ys:

```
verilog_defaults -push
verilog_defaults -add -defer
read_verilog /home/neko/litex/pythondata-cpu-vexriscv/pythondata_cpu_vexriscv/verilog/VexRiscv.v
read_verilog /home/neko/example_project/build/sipeed_tang_nano_9k/gateware/sipeed_tang_nano_9k.v
verilog_defaults -pop
attrmap -tocase keep -imap keep="true" keep=1 -imap keep="false" keep=0 -remove keep=0

synth_gowin  -top sipeed_tang_nano_9k
write_json  sipeed_tang_nano_9k.json
```

<code>sipeed_tang_nano_9k.v</code> isimli dosya tasarımın üst modül kodu. Hataya sebep olan <code>O_psram_reset_n</code> sinyalinin burada yer aldığını görüyoruz.

sipeed_tang_nano_9k.v:

```verilog
// -----------------------------------------------------------------------------
// Auto-Generated by:        __   _ __      _  __
//                          / /  (_) /____ | |/_/
//                         / /__/ / __/ -_)>  <
//                        /____/_/\__/\__/_/|_|
//                     Build your hardware, easily!
//                   https://github.com/enjoy-digital/litex
//
// Filename   : sipeed_tang_nano_9k.v
// Device     : GW1NR-LV9QN88PC6/I5
// LiteX sha1 : 64cf925b3
// Date       : 2024-10-06 17:17:16
//------------------------------------------------------------------------------

`timescale 1ns / 1ps

//------------------------------------------------------------------------------
// Module
//------------------------------------------------------------------------------

module sipeed_tang_nano_9k (
    inout  wire   [15:0] IO_psram_dq,
    inout  wire    [1:0] IO_psram_rwds,
    output reg     [1:0] O_psram_ck,
    output reg     [1:0] O_psram_ck_n,
    output reg     [1:0] O_psram_cs_n,
    output reg     [1:0] O_psram_reset_n,
    (* keep = "true" *)
    input  wire          clk27,
    input  wire          serial_rx,
    output reg           serial_tx,
    output reg           spiflash_clk,
    output wire          spiflash_cs_n,
    input  wire          spiflash_miso,
    output reg           spiflash_mosi,
    input  wire          user_btn0,
    output wire          user_led0,
    output wire          user_led1,
    output wire          user_led2,
    output wire          user_led3,
    output wire          user_led4,
    output wire          user_led5
);

...
```

Kısıt (constraint) dosyasına baktığımızda da psram sinyallerinin burada olmadığı görünüyor.

sipeed_tang_nano_9k.cst:

```
IO_LOC "clk27" 52;
IO_PORT "clk27" IO_TYPE=LVCMOS33;
IO_LOC "user_btn0" 3;
IO_PORT "user_btn0" IO_TYPE=LVCMOS18;
IO_LOC "serial_rx" 18;
IO_PORT "serial_rx" IO_TYPE=LVCMOS33;
IO_LOC "serial_tx" 17;
IO_PORT "serial_tx" IO_TYPE=LVCMOS33;
IO_LOC "spiflash_cs_n" 60;
IO_PORT "spiflash_cs_n" IO_TYPE=LVCMOS33;
IO_LOC "spiflash_clk" 59;
IO_PORT "spiflash_clk" IO_TYPE=LVCMOS33;
IO_LOC "spiflash_miso" 62;
IO_PORT "spiflash_miso" IO_TYPE=LVCMOS33;
IO_LOC "spiflash_mosi" 61;
IO_PORT "spiflash_mosi" IO_TYPE=LVCMOS33;
IO_LOC "user_led0" 10;
IO_PORT "user_led0" IO_TYPE=LVCMOS18;
IO_LOC "user_led1" 11;
IO_PORT "user_led1" IO_TYPE=LVCMOS18;
IO_LOC "user_led2" 13;
IO_PORT "user_led2" IO_TYPE=LVCMOS18;
IO_LOC "user_led3" 14;
IO_PORT "user_led3" IO_TYPE=LVCMOS18;
IO_LOC "user_led4" 15;
IO_PORT "user_led4" IO_TYPE=LVCMOS18;
IO_LOC "user_led5" 16;
IO_PORT "user_led5" IO_TYPE=LVCMOS18;
```

Şimdi bu hataya sebep olan sinyallerin neden ortaya çıktığını inceleyelim. Üzerinde çalıştığımız Python betiği <code>litex-kurulum-dizini/litex-boards/litex_boards/targets/sipeed_tang_nano_9k.py</code> adresinde yer alıyor. Bu dosyaya baktığımızda söz konusu sinyallerin aşağıda verilen kod bloğunda eklendiğini gözlemliyoruz.

sipeed_tang_nano_9k.py:

```Python
...

# HyperRAM ---------------------------------------------------------------------------------
if not self.integrated_main_ram_size:
   # TODO: Use second 32Mbit PSRAM chip.
   dq      = platform.request("IO_psram_dq")
   rwds    = platform.request("IO_psram_rwds")
   reset_n = platform.request("O_psram_reset_n")
   cs_n    = platform.request("O_psram_cs_n")
   ck      = platform.request("O_psram_ck")
   ck_n    = platform.request("O_psram_ck_n")
   class HyperRAMPads:
      def __init__(self, n):
         self.clk   = Signal()
         self.rst_n = reset_n[n]
         self.dq    = dq[8*n:8*(n+1)]
         self.cs_n  = cs_n[n]
         self.rwds  = rwds[n]

   hyperram_pads = HyperRAMPads(0)
   self.comb += ck[0].eq(hyperram_pads.clk)
   self.comb += ck_n[0].eq(~hyperram_pads.clk)
   # FIXME: Issue with upstream HyperRAM core, so use old one. Need to investigate.
   if not os.path.exists("hyperbus.py"):
      os.system("wget https://github.com/litex-hub/litex-boards/files/8831568/hyperbus.py.txt")
      os.system("mv hyperbus.py.txt hyperbus.py")
   from hyperbus import HyperRAM
   self.hyperram = HyperRAM(hyperram_pads)
   self.bus.add_slave("main_ram", slave=self.hyperram.bus, region=SoCRegion(origin=self.mem_map["main_ram"], size=4 * MEGABYTE, mode="rwx"))

...
```

Buradan söz konusu sinyallerin <code>integrated_main_ram_size</code> argümanı verilmediğinde oluştuğu anlaşılıyor.

> :pushpin: Kodda güncel HyperRAM çekirdeğinde problemin olduğu not düşülmüş. Fakat önerilen sürümde de zaten hata alıyoruz. HyperRAM, belki yalnızca Gowin IDE araçlarıyla destekleniyor olabilir.

<code>integrated_main_ram_size</code> argümanına bir değer vererek, derleme işlemini tekrar deniyoruz:

```
$ python3 -m litex_boards.targets.sipeed_tang_nano_9k --integrated-main-ram-size 8192 --toolchain apicula --build
```

Bu defa da aşağıdaki hatayı alıyoruz:

```
Info: Running custom HCLK placer...
Info: Placed 0 cells based on constraints.
Info: Creating initial analytic placement for 7649 cells, random placement wirelen = 287489.
Info:     at initial placer iter 0, wirelen = 3598
Info:     at initial placer iter 1, wirelen = 3727
Info:     at initial placer iter 2, wirelen = 3677
Info:     at initial placer iter 3, wirelen = 3936
Info: Running main analytical placer, max placement attempts per cell = 43319432.
Info:     at iteration #1, type ALU: wirelen solved = 4247, spread = 20743, legal = 21380; time = 0.21s
Info:     at iteration #1, type GSR: wirelen solved = 21380, spread = 21380, legal = 21387; time = 0.20s
Info:     at iteration #1, type DFF: wirelen solved = 21236, spread = 56899, legal = 72407; time = 0.69s
ERROR: Failed to expand region (0, 0) |_> (46, 28) of 10513 LUT4s
0 warnings, 1 error
```

Yosys çıktısındaki tüketim tablosunu incelediğimizde LUT4 kaynağının bir miktar aşıldığı ve üretilen sistemin FPGA kartımıza sığmadığı anlaşılıyor:

```
...

Info: Device utilisation:
Info: 	                 VCC:       1/      1   100%
Info: 	                 IOB:      14/    274     5%
Info: 	                LUT4:   10513/   8640   121%
Info: 	              OSER16:       0/     80     0%
Info: 	              IDES16:       0/     80     0%
Info: 	            IOLOGICI:       0/    276     0%
Info: 	            IOLOGICO:       0/    276     0%
Info: 	           MUX2_LUT5:    3363/   4320    77%
Info: 	           MUX2_LUT6:    1101/   2160    50%
Info: 	           MUX2_LUT7:     426/   1080    39%
Info: 	           MUX2_LUT8:     156/   1080    14%
Info: 	                 ALU:     892/   6480    13%
Info: 	                 GND:       1/      1   100%
Info: 	                 DFF:    2125/   6480    32%
Info: 	           RAM16SDP4:       4/    270     1%
Info: 	               BSRAM:      18/     26    69%
Info: 	              ALU54D:       0/     10     0%
Info: 	     MULTADDALU18X18:       0/     10     0%
Info: 	        MULTALU18X18:       0/     10     0%
Info: 	        MULTALU36X18:       0/     10     0%
Info: 	           MULT36X36:       0/      5     0%
Info: 	           MULT18X18:       0/     20     0%
Info: 	             MULT9X9:       0/     40     0%
Info: 	              PADD18:       0/     20     0%
Info: 	               PADD9:       0/     40     0%
Info: 	                 GSR:       1/      1   100%
Info: 	                 OSC:       0/      1     0%
Info: 	                rPLL:       1/      2    50%
Info: 	           FLASH608K:       0/      1     0%
Info: 	                BUFG:       0/     22     0%
Info: 	                DQCE:       0/     24     0%
Info: 	                 DCS:       0/      8     0%
Info: 	              CLKDIV:       0/      8     0%
Info: 	             CLKDIV2:       0/     16     0%

...
```

<code>-\-cpu-type</code> argümanı verilmediği takdirde varsayılan işlemci olarak Vexriscv seçiliyor. Vexriscv, görece yüksek performanslı bir işlemci. Bu noktada daha küçük alan kaplayan işlemcileri deneyebiliriz.

Ibex işlemcisini deniyoruz:

```
$ python3 -m litex_boards.targets.sipeed_tang_nano_9k --cpu-type ibex --integrated-main-ram-size 8192 --toolchain apicula --build
```

Bu sefer aşağıdaki hatayı alıyoruz:

```
2. Executing Verilog-2005 frontend: /home/neko/litex/pythondata-cpu-ibex/pythondata_cpu_ibex/system_verilog/vendor/lowrisc_ip/ip/prim/rtl/prim_alert_pkg.sv
Parsing SystemVerilog input from `/home/neko/litex/pythondata-cpu-ibex/pythondata_cpu_ibex/system_verilog/vendor/lowrisc_ip/ip/prim/rtl/prim_alert_pkg.sv' to AST representation.
/home/neko/litex/pythondata-cpu-ibex/pythondata_cpu_ibex/system_verilog/vendor/lowrisc_ip/ip/prim/rtl/prim_alert_pkg.sv:19: ERROR: syntax error, unexpected OP_CAST
```

Ibex çekirdeği SystemVerilog ile yazıldığı için ne yazık ki Yosys tarafından desteklenmiyor.

Tang Nano 9K FPGA için kullanacağımız işlemci hem küçük hem de Verilog ile yazılmış olmalı. Bu noktada Picorv32, Serv ve FemtoRV gibi çekirdekleri deneyebiliriz. Ben Picorv32'yi seçiyorum:

```
$ python3 -m litex_boards.targets.sipeed_tang_nano_9k --cpu-type picorv32 --integrated-main-ram-size 8192 --toolchain apicula --build
```

Nihayet bu sefer hata almıyoruz:

```
Info: Program finished normally.
```

Derleme ve sentez işlemi başarılı olduğuna göre artık Bitstream'i FPGA'ya yükleyebiliriz. Bunun için <code>-\-load</code> argümanını kullanıyoruz. Kalıcı olarak yüklemek için <code>-\-flash</code> opsiyonuyla Flash belleğe yazabiliriz.

```
$ python3 -m litex_boards.targets.sipeed_tang_nano_9k --cpu-type picorv32 --integrated-main-ram-size 8192 --toolchain apicula --build --load
```

Yükleme tamamlandıktan sonra ürettiğimiz sistemin Litex BIOS'a erişip erişmediğini kontrol edeceğiz. Bunun için Litex Terminal uygulamasını kullanabiliriz. Litex Terminal argüman olarak seri bağlantı yolunu alıyor. Opsiyonel olarak da Baudrate belirtiyoruz:

```
$ litex_term /dev/ttyUSB1 --speed 115200
```

Terminal açıldıktan sonra çıktıyı gözlemlemek için işlemciyi sıfırlamalıyız. Reset butonu, FPGA kartı üzerinde S2 olarak işaretlenmiş.

Litex BIOS çıktısı:

```
        __   _ __      _  __
       / /  (_) /____ | |/_/
      / /__/ / __/ -_)>  <
     /____/_/\__/\__/_/|_|
   Build your hardware, easily!

 (c) Copyright 2012-2024 Enjoy-Digital
 (c) Copyright 2007-2015 M-Labs

 BIOS built on Oct  6 2024 17:56:26
 BIOS CRC passed (be4fa310)

 LiteX git sha1: 64cf925b3

--=============== SoC ==================--
CPU:		   PicoRV32 @ 27MHz
BUS:		   wishbone 32-bit @ 4GiB
CSR:		   32-bit data
ROM:		   64.0KiB
SRAM:		   8.0KiB
FLASH:		   4.0MiB
MAIN-RAM:	   8.0KiB

--========== Initialization ============--

Memtest at 0x40000000 (8.0KiB)...
  Write: 0x40000000-0x40002000 8.0KiB   
   Read: 0x40000000-0x40002000 8.0KiB   
Memtest OK
Memspeed at 0x40000000 (Sequential, 8.0KiB)...
  Write speed: 724.0KiB/s
   Read speed: 723.4KiB/s

Initializing w25q32 SPI Flash @0x00000000...
SPI Flash clk configured to 13 MHz
Memspeed at 0 (Sequential, 4.0KiB)...
   Read speed: 281.9KiB/s
Memspeed at 0 (Random, 4.0KiB)...
   Read speed: 58.3KiB/s

--============== Boot ==================--
Booting from serial...
Press Q or ESC to abort boot completely.
sL5DdSMmkekro
Timeout
No boot medium found

--============= Console ================--

litex> help

LiteX BIOS, available commands:

leds                     - Set Leds value
flush_cpu_dcache         - Flush CPU data cache
crc                      - Compute CRC32 of a part of the address space
ident                    - Identifier of the system
help                     - Print this help

serialboot               - Boot from Serial (SFL)
reboot                   - Reboot
boot                     - Boot from Memory

mem_cmp                  - Compare memory content
mem_speed                - Test memory speed
mem_test                 - Test memory access
mem_copy                 - Copy address space
mem_write                - Write address space
mem_read                 - Read address space
mem_list                 - List available memory regions

litex> 

```

Çıktıda görüldüğü gibi BIOS, öncelikle seri önyüklemeyi deniyor. Seri port üzerinden herhangi bir program göndermediğimizde, birkaç komut destekleyen bir konsol açılıyor.

Bu noktada BIOS'tan bir programı önyüklemeyi deneyebiliriz. Litex projesinde, oluşturduğumuz sistem için derleyebileceğimiz "bare metal demo app" isminde örnek bir program bulunuyor. Bu programı derlemek için <code>litex_bare_metal_demo</code> komutunu giriyoruz. Bu komuta argüman olarak "build" dizinimizi göstermeliyiz.

```
$ litex_bare_metal_demo --build-path=/home/neko/example_project/build/sipeed_tang_nano_9k
```

Komut yürütüldüğünde çıktısı şu şekilde:

```
CC       donut.o
CC       helloc.o
CC       crt0.o
CC       main.o
CC       demo.elf
chmod -x demo.elf
OBJCOPY  demo.bin
chmod -x demo.bin
```

Örnek proje derlendi ve sisteme yüklenmeye hazır. İkilik dosyayı seri port üzerinden göndermek için aşağıdaki komutu giriyoruz.

```
$ litex_term /dev/ttyUSB1 --speed 115200 --kernel=demo.bin
```

Uçbirim açılıp, işlemciyi resetlediğimizde tekrar hata alıyoruz:

```
--============== Boot ==================--
Booting from serial...
Press Q or ESC to abort boot completely.
sL5DdSMmkekro
[LITEX-TERM] Received firmware download request from the device.
[LITEX-TERM] Uploading demo.bin to 0x40000000 (8132 bytes)...
[LITEX-TERM] Upload calibration... (inter-frame: 10.00us, length: 64)
[LITEX-TERM] Got unexpected response from device 'b'E''
```

Araştırmalarım sonucunda hataya neyin sebep olduğu ve çözümü hakkında net bir bilgiye ulaşamadım ancak Litex Terminal'in <code>-\-safe</code> opsiyonu dikkatimi çekti.

```
--safe  Safe serial boot mode, disable upload speed optimizations.
```

Güvenli seri önyükleme modu ile tekrar deniyoruz:

```
$ litex_term /dev/ttyUSB1 --speed 115200 --kernel=demo.bin --safe
```

Bu komutla birlikte seri önyükleme işlemini başarıyla gerçekleştirebildik. Bu noktada önceki komutta aldığımız hatanın sebebini düşük çalışma frekansına bağlayabilmemiz mümkün.

```
        __   _ __      _  __
       / /  (_) /____ | |/_/
      / /__/ / __/ -_)>  <
     /____/_/\__/\__/_/|_|
   Build your hardware, easily!

 (c) Copyright 2012-2024 Enjoy-Digital
 (c) Copyright 2007-2015 M-Labs

 BIOS built on Oct  6 2024 20:03:48
 BIOS CRC passed (91a777bd)

 LiteX git sha1: 64cf925b3

--=============== SoC ==================--
CPU:		PicoRV32 @ 27MHz
BUS:		wishbone 32-bit @ 4GiB
CSR:		32-bit data
ROM:		64.0KiB
SRAM:		8.0KiB
FLASH:		4.0MiB
MAIN-RAM:	8.0KiB

--========== Initialization ============--

Memtest at 0x40000000 (8.0KiB)...
  Write: 0x40000000-0x40002000 8.0KiB   
   Read: 0x40000000-0x40002000 8.0KiB   
Memtest OK
Memspeed at 0x40000000 (Sequential, 8.0KiB)...
  Write speed: 724.0KiB/s
   Read speed: 723.4KiB/s

Initializing w25q32 SPI Flash @0x00000000...
SPI Flash clk configured to 13 MHz
Memspeed at 0 (Sequential, 4.0KiB)...
   Read speed: 281.9KiB/s
Memspeed at 0 (Random, 4.0KiB)...
   Read speed: 58.3KiB/s

--============== Boot ==================--
Booting from serial...
Press Q or ESC to abort boot completely.
sL5DdSMmkekro
[LITEX-TERM] Received firmware download request from the device.
[LITEX-TERM] Uploading demo.bin to 0x40000000 (8132 bytes)...
[LITEX-TERM] Upload complete (2.5KB/s).
[LITEX-TERM] Booting the device.
[LITEX-TERM] Done.
Executing booted program at 0x40000000

--============= Liftoff! ===============--

LiteX minimal demo app built Oct  6 2024 18:29:45

Available commands:
help               - Show this command
reboot             - Reboot CPU
led                - Led demo
donut              - Spinning Donut demo
helloc             - Hello C
litex-demo-app> donut
Donut demo...

                                                                               
                                                                               
                                                                               
                                  @@@$$@$$$@@$                                 
                             $$$$$###########$$$$$                             
                           #####**!!!!!!!!!!!**###$$#                          
                         ####**!==!==========!!!**####                         
                        ###*!!!!==;;:::::::;;=!!!!!**##*                       
                       ****!!!!=::~--,,.,,-~~;;=!=!!*****                      
                      ****!!==;;:-,.........,~:;==!!!****!                     
                     =***!!!!=;:~,...........-~:;=!!!!***!                     
                     !!**!!!=;;:-,..       ..,-:;==!!!**!!=                    
                     =!!*!!!==;:~,.         ~;==!!!!**!!!=;                    
                     ;!!!****!***!==       ;!!!!******!!!=:                    
                     :=!!!*****#######***######******!!!=;                     
                      ;=!!***####$$$$$$@$$$$$###*****!!=;:                     
                       :==!!***##$$$$@@@@@$$$###***!!!=;~                      
                        ~;=!!**#####$$$$$$$###****!!==;-                       
                         ,:;=!!!*!*#########!**!!!=;;~.                        
                           -~:;==!!!*******!!!==;;:~.                          
                              ,-::;;=======;;;:~-,                             
                                 ..,,------,,.                                 


```

Demo kodunda, "Hello World" uygulaması <code>helloc</code>, kartın üzerindeki LED'lerle ışık gösterisi yapan <code>led</code> ve uçbirimde ASCII karakterlerle hareket eden <code>donut</code> programları bulunuyor.
