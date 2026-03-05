---
title: "Harika Açık Kaynaklı Sayısal Tasarım"
date: 2026-03-05
description: Sayısal tasarımcıların işine yarayabilecek bazı proje ve araçların listesi 
math: true
---

<p></p>

<p align="center">
  <img src="/awesome_dd.png"/>
</p>

GitHub'da [Awesome](https://github.com/topics/awesome) listeleri oldukça popüler. Bende tarayıcımın yer imlerine kaydettiğim projelerin hayli çoğaldığını görünce bir liste hazırlamak istedim.

Bu listede yer alması gerektiğini düşündüğünüz başka projeler varsa yorumlarda belirtebilirsiniz.

**RISC-V**

* [PicoRV32](https://github.com/YosysHQ/picorv32) - Küçük ve yaygın kullanılan RISC-V işlemci
* [NEORV32](https://github.com/stnolting/neorv32) - VHDL ile yazılmış mikrodenetleyici benzeri RISC-V projesi
* [Hazard3](https://github.com/Wren6991/Hazard3) - Raspberry Pi RP2350'nin içinde bulunan RISC-V çekirdeği
* [Ibex](https://github.com/lowRISC/ibex) - LowRISC tarafından geliştirilen küçük RISC-V işlemci
* [VexRiscv](https://github.com/SpinalHDL/VexRiscv) - Hızlı ve FPGA için optimize RISC-V işlemci
* [VexiiRiscv](https://github.com/SpinalHDL/VexiiRiscv) - Daha gelişmiş VexRiscv
* [NaxRiscv](https://github.com/SpinalHDL/NaxRiscv) - Sırasız yürütüm yapan RISC-V işlemci
* [SCR1](https://github.com/syntacore/scr1) - Mikrodenetleyici sınıfı RISC-V işlemci
* [CV32E40P](https://github.com/openhwgroup/cv32e40p) - OpenHW Group tarafından geliştirilen 4 aşama boru hatlı işlemci
* [CVA5](https://github.com/openhwgroup/cva5) - OpenHW Group tarafından geliştirilen 5 aşama boru hatlı işlemci
* [CVW](https://github.com/openhwgroup/cvw) - RISC-V System-on-Chip Design kitabında anlatılan işlemci
* [CVA6](https://github.com/openhwgroup/cva6) - OpenHW Group tarafından geliştirilen 6 aşama boru hatlı işlemci
* [VeeR](https://github.com/chipsalliance/Cores-VeeR-EH2) - Western Digital'in RISC-V işlemci tasarımı (Eski adı: SweRV)
* [Toooba](https://github.com/bluespec/Toooba) - MIT tarafından geliştirilen sırasız yürütüm yapan işlemci
* [BOOM](https://github.com/riscv-boom/riscv-boom) - Berkeley tarafından geliştirilen sırasız yürütüm yapan işlemci
* [XiangShan](https://github.com/OpenXiangShan/XiangShan) - Çin menşeili yüksek performanslı işlemci

**Benchmark ve Testler**

* [Dhrystone](https://github.com/sifive/benchmark-dhrystone) - Meşhur performans ölçüm programı
* [CoreMark](https://github.com/eembc/coremark) - Gömülü sistemler için performans ölçüm programı
* [Embench](https://github.com/embench/embench-iot) - FOSSi derneği tarafından geliştirlen modern benchmark
* [bringup-bench](https://github.com/toddmaustin/bringup-bench) - İşlemci testi için çeşitli programlar
* [TinyPrograms](https://github.com/BrunoLevy/TinyPrograms) - Demoscene tarzı programlar

**TPU ve NPU**

* [Coral](https://github.com/google-coral/coralnpu) - Google tarafından geliştirilen NPU
* [NVDLA](https://github.com/nvdla/hw) - Nvidia tarafından geliştirilen derin öğrenme hızlandırıcı
* [Gemmini](https://github.com/ucb-bar/gemmini) - Berkeley tarafından geliştirilen yapay sinir ağı hızlandırıcı
* [FPGA NPU](https://github.com/intel/fpga-npu) - Intel tarafından geliştirilen NPU

**GPU ve GPGPU**

* [Miaow](https://github.com/VerticalResearchGroup/miaow) - Radeon komut setini referans alan GPU tasarımı
* [Radiance](https://github.com/ucb-bar/radiance) - Berkeley tarafından geliştirilen GPU mimarisi
* [Vortex](https://github.com/vortexgpgpu/vortex) - RISC-V komut setli GPGPU
* [Ventus](https://github.com/THU-DSP-LAB/ventus-gpgpu) - Chisel ile yazılmış GPGPU
* [gplgpu](https://github.com/asicguy/gplgpu) - [Number Nine](https://en.wikipedia.org/wiki/Number_Nine_Visual_Technology) mimarili GPU

**FPGA Generator**

* [OpenFPGA](https://github.com/lnis-uofu/OpenFPGA) - FPGA IP üretici
* [FABulous](https://github.com/FPGA-Research/FABulous) - FPGA IP üretici
* [PRGA](https://github.com/PrincetonUniversity/prga) - FPGA IP üretici

**System on Chip**

* [Litex](https://github.com/enjoy-digital/litex) - Hızlı ve kullanımı kolay SoC oluşturma aracı
* [Chipyard](https://github.com/ucb-bar/chipyard) - Berkeley tarafından geliştirilen SoC oluşturma aracı
* [Core-V MCU](https://github.com/openhwgroup/core-v-mcu) - CV32E40P işlemcisini içeren mikrodenetleyici sınıfı SoC
* [Veerwolf](https://github.com/chipsalliance/VeeRwolf) - Veer işlemcileri ile konfigüre edilebilen SoC
* [Ibex Demo System](https://github.com/lowRISC/ibex-demo-system) - Ibex işlemcisi ile hazırlanmış küçük SoC
* [OpenTitan](https://github.com/lowRISC/opentitan) - LowRISC tarafından geliştirilen referans sistem
* [Caliptra](https://github.com/chipsalliance/caliptra-rtl) - CHIPS Alliance tarafından geliştirilen referans sistem

**HDL ve HLS**

* [Chisel](https://github.com/chipsalliance/chisel) - Scala tabanlı yüksek seviyeli HDL
* [Bluespec](https://github.com/B-Lang-org/bsc) - Yüksek seviyeli HDL
* [ROHD](https://github.com/intel/rohd) - Intel tarafından geliştirilen Dart tabanlı donanım oluşturucu araç
* [XLS](https://github.com/google/xls) - Google tarafından geliştirilen Rust benzeri sözdizime sahip HLS
* [Kanagawa](https://github.com/microsoft/kanagawa) - Microsoft tarafından geliştirilen HLS
* [Amaranth](https://github.com/amaranth-lang/amaranth) - Python tabanlı HDL
* [SpinalHDL](https://github.com/SpinalHDL/SpinalHDL) - Scala tabanlı yüksek seviyeli HDL

**Sentez ve Benzetim**

* [Yosys](https://github.com/YosysHQ/yosys) - Birçok farklı mimariyi destekleyen açık kaynaklı sentez aracı
* [Iverilog](https://github.com/steveicarus/iverilog) - Verilog için derleyici ve simülatör
* [GHDL](https://github.com/ghdl/ghdl) - VHDL için derleyici ve simülatör
* [Verilator](https://github.com/verilator/verilator) - Verilog ve SystemVerilog için simülatör
* [cocotb](https://github.com/cocotb/cocotb) - Python tabanlı doğrulama aracı
* [Verible](https://github.com/chipsalliance/verible) - SystemVerilog için geliştirici araçları
* [Surelog](https://github.com/chipsalliance/Surelog) - SytemVerilog için geliştirici araçları
* [Slang](https://github.com/MikePopoloski/slang) - SystemVerilog için geliştirici araçları
* [Yosys-Slang](https://github.com/povik/yosys-slang) - Yosys için SystemVerilog eklentisi
* [Synlig](https://github.com/chipsalliance/synlig) - Yosys ile uyumlu SystemVerilog sentez aracı
* [sv2v](https://github.com/zachjs/sv2v) - SystemVerilog - Verilog dönüştürücüsü
* [vhd2vl](https://github.com/ldoolitt/vhd2vl) - VHDL - Verilog dönüştürücüsü
* [GTKWave](https://github.com/gtkwave/gtkwave) - GTK+ tabanlı dalga formu görüntüleyicisi
* [Surfer](https://gitlab.com/surfer-project/surfer) - Dalga formu görüntüleyicisi
* [Vaporview](https://github.com/Lramseyer/vaporview) - VS Code için dalga formu görüntüleyici eklentisi
* [vcdrom](https://github.com/wavedrom/vcdrom) - Javascript ile yazılmış dalga formu görüntüleyicisi

**Dokümantasyon**

* [wavedrom](https://github.com/wavedrom/wavedrom) - Zamanlama diyagramı üretici
* [bitfield](https://github.com/wavedrom/bitfield) - Bit alanı diyagramı üretici
* [schemdraw](https://github.com/cdelker/schemdraw) - Elektronik şema üretici
* [netlistsvg](https://github.com/nturley/netlistsvg) - Yosys JSON çıktısı için şema üretici
* [symbolator](https://github.com/hdl/symbolator) - VHDL ve Verilog kodundan şema üretici
* [Konata](https://github.com/shioyadan/Konata) - Boru hattı komut akışı görselleştirici

**Benzer Listeler**

* [Awesome Open Source Hardware](https://github.com/aolofsson/awesome-opensource-hardware)
* [Awesome Open Source ASIC Resources](https://github.com/mattvenn/awesome-opensource-asic-resources)
* [Awesome HDL](https://hdl.github.io/awesome/items/)
