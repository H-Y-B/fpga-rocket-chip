# Step-by-step Guide of Booting linux on a Rocket-chip SoC on Nexys4ddr

## I. Big Map

The main idea of this guide is to provide a step-by-step tutorial of building a **RISC-V** SoC, especially for those who are interested in the **Rocket-chip** and want to test it on a real board instead of simulators. At the end of the tutorial, you will be able to boot a Linux on the **Nexys4ddr** FPGA board and run your own riscv programs on it.

A general background is provided by the following links: [RISC-V](https://riscv.org) and [Rocket-Chip](https://github.com/freechipsproject/rocket-chip).

### 1.1 How a Linux gets booted

![system structure](pics/bigmap.png)

The **ExampleRocketSystem** generated by Rocket-chip is the top module that we will adopt to build our system. It exposes a MIMO-AXI4 and a Mem-AXI4 interfaces to the outside peripherals, of which the former will be connected to IO devices including **Uart**(@0x60000000), **SDslot**(@0x60020000) and **BRAM_64K**(@0x60010000) and the latter will be used to connect **DDR**(@0x80000000) memory.

Here is the whole story of the booting procedure: 

- After power on, the PC (program counter) is set to 0x10040 according to the variable **Reset_Vector**. The processor starts executing  instructings in **TLBootrom**(@0x10000) inside of ExampleRocketChip. TLBootrom contains instrcutions asking the processor jumping to BRAM_64K: `li s0, BRAM_BASE`, ` jr s0`.
- In the **BRAM_64K**, two tasks of our **first stage bootloader** (aka FSBL) are accomplished: to copy the elf image, which contains a **BBL** (Berkeley bootloader) and a **vmlinux** image to somewhere (@0x870000000) in the DDR, and to extract this elf onto the beginning of DDR(@0x80000000). Following the two tasks, the FSBL will jump to DDR and the PC will move to the bbl:  `li s0, DDR_BASE`, ` jr s0`.
- In the DDR, BBL will do some preparation works, including setting traps, SBI (supervisor binary interface) and physical memory protection, and then handing over a DTB (device tree blob) to the kernel (in early versions, BBL also virtualizes the memory, which is now directly handled by the Linux). Ultimately, BBL starts the Linux kernel: `enter_supervisor_mode(entry, hartid, dub_output())`.
- After the initialization of the kernel, **busybox_init** starts and calls the **busybox_ash**, which provides you with a command line interface.

### 1.2 Prerequisites

You should know the followings before continue to the next section.

- how to use and read the GNU Makefiles
- how to compile C/C++ using GNU GCC
- C and Assembly language
- Basic knowledge of compiling a Linux kernel
- Some experience in designing hardware using Verilog/SystemVerilog

### 1.3 Preparing the Project

Download the source files:

`git clone -b nexys4-demo --recursive https://github.com/cnrv/fpga-rocket-chip`

There are several dirs in the repo:

- **constraints**, **verilog**, **firmware** - those are sources that we will use to build Vivado project and FPGA configration file, namely, mcs. Firmware contains the SDloader program (firmware.hex) that load elf image into DDR, functioning as a FSBL, and it will be burned into BRAM_64K.
- **riscv-pk** - a modified version of [riscv/riscv-pk](https://github.com/riscv/riscv-pk) repo.  BBL together with linux kernel will be put into SDcard and get loaded to DDR by FSBL. **NOTICE:** We implant the DTB into the bbl and slightly change the uart driver. Generally DTB should be located in the firmware, but for debugging convenience (it is time-consuming to change the firmware and reburn it into FPGA), I put it just inside the bbl. DTS is located at riscv-pk/build.
- **rocket-chip** - a modified version of [freechipsproject/rocket-chip](https://github.com/freechipsproject/rocket-chip) repo. We changed the Bootrom (TLBootrom)  content, so that the cpu can jump to BRAM_64K once it is powerd on.
- **pics** - pictures for this markdown file

Besides, you also need to download the following repos:

- **cross compiler** - `git clone https://github.com/riscv/riscv-gnu-toolchain` 
  - hash 055959d1334b9d3c3cfbabbfe586241ce0edaf5c , others should also work
  - install **elf-gcc and** **linux-gcc** and set **RISCV** variable in advance
- **linux kernel** -  ` git clone https://github.com/riscv/riscv-linux` 
  - hash 8fe28cb58bcb235034b64cbbb7550a8a43fd88be , others should also work
- **busybox** that provides init and utils - `git clone https://github.com/mirror/busybox` 
  - version 1.30 stable , others should also work

## II. Hardware generation 

**Vivado** version 2016.4; **Ubuntu** version 16 LTS

The tutorial should work well for most version of vivado.

Also, do not forget to install **vivado usb driver**, in case the Hardware Manager cannot recognize the board.

### 2.1 building the vivado project

#### 2.1.1 generate source files

- `cd fpga-rocket-chip`
- `make vivado_source`
- then you should get **firmware.hex** and **DefaultConfig.v** under /verilog directory.
- together with other verilog files, they are the source files used to build vivado project.

#### 2.1.2 new project wizard

- Creat New Project 
- Add Sources - Add Directories - choose **/verilog**
- Add Constraints - Add Files - choose **/constraints/Board_Pin_Map.xdc**
- Default Part - Parts - choose **xc7a100tcsg324-1** , this is the chip that nexys4ddr holds.
- In the **Project Manager** window, right click **dut_inst - rocketTop (chip_top.v)** and set it as Top module

#### 2.1.3 Adding Peri IPs

Before I output the TCL script (one of my future work),  at current stage please add those ips manually. Here's the configuration list.

**a) clocking wizard** 

- Component name - clk_wiz_0
- Clocking Options - Primitive - **PL**L
- Output Clocks - Output Clock - **clk_out1 30.000; clk_out2 200.000**
- Output Clocks - Enable Optional IO - check **reset and locked**
- Output Clocks - Reset Type - **Active Low**

**b) AXI crossbar**

- Component name - axi_crossbar_0
- Global: Num of Slave - **1**; Num of Master - **3**; Protocol - **AXI4**
- Global: Addr Width - **31**; Data Width - **64**; ID Width - **4**
- Address: M00_AXI - **0x60000000 - 13**; M01_AXI - **0x60010000 - 16**; M00_AXI - **0x60020000 - 7**

**c) AXI UART16550**

- [doc](https://www.xilinx.com/support/documentation/ip_documentation/axi_uart16550/v2_0/pg143-axi-uart16550.pdf)
- Component name - axi_uart16550_0
- AXI CLK Frequency - **30 MHz**
- UART MOde - **16550**

**d) AXI BRAM Controller**

- Component name - axi_bram_ctrl_0
- AXI protocol - **AXI4**; Data Width - **64**; Memory Depth - **8192**; ID Width - **4**
- BRAM_INSTANCE - **External**; Num of BRAM interface - **1**
- Enable ECC - **no**

**e) AXI Quad SPI**

- [doc](https://www.xilinx.com/support/documentation/ip_documentation/axi_quad_spi/v3_2/pg153-axi-quad-spi.pdf)
- Component name - axi_quad_api_0
- AXI Interface Options - **uncheck all**
- SPI options: Mode - **standard**; Transaction width - **8**; Frequency Ration - **2x1**; Num of Slaves - **1**
- Check Enable Master Mode; Check **Enable FIFO** - FIFO Depth **16**

**f) AXI Clock Converter**

- Component name - axi_clock_converter_0
- Protocol - **AXI4**; Read_Write Mode - **read write**; Addr Width - **32**; Data Width - **64**; ID Width - **4**
- Clock Concersion Options - **Asynchronous - Yes**

#### 2.1.4 DDR controller 

- [doc](https://www.xilinx.com/support/documentation/ip_documentation/mig_7series/v4_2/ds176_7Series_MIS.pdf)
- IP Catalog - Memory Interface Generator
- Component name - mig_7series_0
- MIG output options - Creat Design
- Check **AXI4 Interface**
- Pin Compatible FPGA - Select **xc7a100ti-csd324**
- Memory Selection - **DDR2 SDRAM**
- Options for Controller: Clock Period - **5000ps**; PHY to Controller Clock Ration - **4:1**
- Options for Controller: Memory Part - **MT47H64M16HR-25E**; Data Width - **16**;
- Options for Controller: Num of Bank Machines - **4**; Ordering - **Normal**
- AXI Parameter Options: Data Width - **64**; Arbitration Scheme - **RD_PRI_REG**; ID Width: **4**
- Memory Options for Controller: Input Clock Period - **5000ps (200MHz)**
- Memory Options for Controller: Burst type - **Sequential**; Output Drive Strength - **Full**
- Memory Options for Controller: Controller Chip Select Pin - **Enable**; ODT - **50ohms**
- Memory Options for Controller: Memory Address Mapping Selection - **BANK/ROW/COLUMN**
- System Clock - No Buffer; Reference Clock - **Use System Clock**
- System Reset Polarity - **Active HIGH**
- Debug Signal - off; check Internal Verf; IO Power Reduction - off; XADC - off
- Internal Termination Impedance - **50ohms**
- Pin/Bank Selection Mode - **Fixed Pin Out**
- Pin Selection For Controller - Read XDC/UCF - select **/constraints/DDR_Pin_Map.ucf** - Validate
- Accept

### 2.2 generating MCS

- Generate BitStream
- This may take a long time: core i7-6700, 5min for synthesis, 7min for implementation
- Tools - Generate Memory Configuartion File
  - Format - **MCS**; Memory Part - **s25fl128sxxxxxx0-spi**; Interface - **SPIx1**
  - Do not forget to specify a filename and the output path
  - Check Load bitstream files - select  **your_project_dir/.runs/impl_1/.bit**

- Then you can find a ***.mcs** file under the path you specify in the Filename Option.
- Hardware Manager - add configuration memory device - **s25fl128sxxxxxx0-spi**
- Hardware Window - program configuration memory device
  - select your ***.mcs** file as configuration file
  - Apply

- Or if you wish, you could use **Program Device** that sending bitstream to FPGA through USB directly, in which case, you need to program FPGA manually everytime you poweroff it.

## III. linux Image generation

### 3.1 build busybox 

Busybox is a useful binary tool, or a tools set to be exact. You can integrate many frequently used commands (e.g. cd, ls, echo) into this single program. Our **init** is also provided by busybox program.

- download busybox, by github or by busybox.net both works
- `cd busybox-1.xx.xx` 
- `make allnoconfig`  - to turn off every configuration option
- `make menuconfig`
- Here are the configurations you will have to change:
  - `CONFIG_STATIC=y`, **Build BusyBox as a static binary (no shared libs)** in **Settings - Build Options** 
  - `CONFIG_CROSS_COMPILER_PREFIX=riscv64-unknown-linux-gnu-`, **Cross Compiler prefix** in **Settings - Build Options** 
  - `CONFIG_FEATURE_INSTALLER=y`, **Support --install [-s] to install applet links at runtime** in **Settings - General Configuration - include busybox applet**
  - `CONFIG_INIT=y`, **init** in **Init utilities**
  - `CONFIG_ASH=y`, **ash** in **Shells**
  - `CONFIG_MOUNT=y`, **mount** in **Linux System Utilities**
  - `CONFIG_FEATURE_USE_INITTAB=y`, **Support reading an inittab file** in **Init Utilities**
  - (this section is in reference of riscv-tools/README.md)

- Besides the above, you can optional check the following tools in the menuconfig, according to your needs:
  - **Coreutils** - cat / chmod / echo / ls / mkdir / mknod / pwd / rm / tty
  - **Linux System Utilities** - fdisk / umount 

- For your convenience, there is a **config_busybox** under the **config** directory, you can copy it and use it to overwrite the **.config** under the busybox dir
- `make -jN`  (N is the max number of your parallel jobs)
- Then you will have a **busybox** over there.
- an **inittab** is needed to instruct the init program, create a document named as **inittab** and write the following into it:
  - ::sysinit:/bin/busybox mount -t proc proc /proc
  - ::sysinit:/bin/busybox mount -t tmpfs tmpfs /tmp
  - ::sysinit:/bin/busybox mount -o remount,rw /
  - ::sysinit:/bin/busybox --install -s
  - /dev/console::sysinit:-/bin/ash

### 3.2 set up initramfs

We gonna build the file system that stores in the elf image, it will be extract as the root file system when linux is booted.

- create files and dirs
  - ` mkdir root`
  - `cd root`
  - `mkdir -p bin etc dev lib proc sbin sys tmp usr usr/bin usr/lib usr/sbin`
- Then copy our **busybox** we have built into the filesystem
  - `cp <your_busybox> bin/busybox`
  - make soft links so that busybox can work as the **init** program
  - `ln -s bin/busybox sbin/init`
  - `ln -s bin/busybox init`
- copy **inittab** you created above into the /etc
  - `cp <your_inittab_file> etc/inittab`
- create a character device for the console
  - `sudo mknod dev/console c 5 1`
- compress it into a cpio file
  - `find . | cpio --quiet -o -H newc | gzip > ../rootfs.cpio.gz`
  - `cd ..`
  - then you will find a **rootfs.cpio.gz** over there, next to the dir **root** 
- (this section is in reference of riscv-tools/README.md)

### 3.3 compiling Linux

Now we have our initramfs, **rootfs.cpio.gz**. We can build the linux kernel now.

- copy the rootfs.cpio.gz to the riscv-linux dir and go into riscv-linux
- `cp <your_rootfs.cpio.gz> <path_to_your_riscv-linux>`
- `cd <path_to_your_riscv-linux>`
- `make ARCH=riscv defconfig`
- `make ARCH=riscv menuconfig`
- the very important change to the defconfig is to enable the initramfs here
  - in the **General Setup**
  - check **initial ram filesystem and ram disk (initramfs/initrd) support**
  - fill **rootfs.cpio.gz** in the blank blow
  - ![](pics/kernelconfig.png)
  - For your convenience, there is a **config_linux** under the **config** directory, you can copy it and use it to overwrite the **.config** under the linux dir

- `make -jN ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- vmlinux`
- after several  minute, there will be a **vmlinux** under the dir of riscv-linux

### 3.4 build the image

Once you have compile your own linux kernel, vmlinux. Then it comes to the building of the sd image. Note down the path of your vmlinux, so that the compiler can link bbl with your vmlinux.

- `cd fpga-rocket-chip`
- `make sd_image VMLINUX=<path_of_your_vmlinux>`
- you will find a **boot.elf** over there 

## IV On board experiment

- format your sdcard with fat32
- drag your **boot.elf** into the sdcard
- eject the sd and insert it into the SDslot on Nexys4ddr board
- connect the usb and power on the board
- use serial tools like minicom to capture the output from the board, 115200 8n1
- Hooray~~~
- ![](pics/minicom.png)
- here I put a static-compiled helloworld program inside of the rootfs.cpio, you can also put your own riscv program inside. However the final boot.elf cannot larger than 16MB (cause the sd_loader copy boot.elf to 0x87000000, only 2^24 Bytes to hold the image).

## V. Q&A, future map, known bugs

TBA...

## VI. Acknowledgement

TBA...

Prof. Wei Song

lowrisc

bar-riscv

riscv-tools



