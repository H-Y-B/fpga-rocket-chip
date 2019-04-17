# Step-by-step Guide of Booting linux on a Rocket-chip SoC on Nexys4ddr

## I. Big Map

The main idea of this guide is to provide a step-by-step tutorial of building a **RISC-V** SoC, especially for those who are interested in the **Rocket-chip** and want to test it on a real board instead of simulators. At the end of the tutorial, you will be able to boot a Linux on the **Nexys4ddr** fpga board and run your own riscv programs on it.

A general background is provided by the following links: [RISC-V](https://riscv.org) and [Rocket-Chip](https://github.com/freechipsproject/rocket-chip).

### 1.1 How the linux gets booted

![system structure](pics/bigmap.png)

The **ExampleRocketSystem** generated by Rocket-chip is the top module that we will adopt to build our system. It exposes a MIMO-AXI4 and a Mem-AXI4 interfaces to the outside peripherals, of which the former will be connected to IO devices including **Uart**(@0x60000000), **SDslot**(@0x60020000) and **BRAM_64K**(@0x60010000) and the later will be used to connect **DDR**(@0x80000000) memory.

Here is the whole story of the booting procedure: 

- Power on and PC is set to 0x10040 according to the variable **Reset_Vector**. The cpu starts executing  instructings in **TLBootrom**(@0x10000) inside of ExampleRocketChip. TLBootrom contains instrcutions asking the processor jumping to BRAM_64K: `li s0, BRAM_BASE`, ` jr s0` 
- In the **BRAM_64K**, two tasks of our **first stage bootloader** (aka FSBL) are accomplished:  to copy the elf image (which contains bbl and vmlinux) to  somewhere(@0x870000000) in the DDR, to extract this elf onto the beginning of DDR(@0x80000000). Following the two tasks, the FSBL will jump to DDR and PC wil move to the bbl:  `li s0, DDR_BASE`, ` jr s0` 
- In the **DDR**, **BBL**(aka BerkeleyBootloader) will do some preparation works including setting traps and SBI, physical memory protection and hand over the DTB to the kernel (in the early version, it also virtualizes the memory. Now linux can do this job on its own.).  Following BBL, starts the linux kernel: `enter_supervisor_mode(entry, hartid, dub_output());`
- After the initialization of the kernel, **busybox_init** starts and it will call the **busybox_ash**, then you can interact with the kernel.

### 1.2 basic knowledge required

- Make and Makefile
- How codes turn into machine instructions
- C and Assembly language
- Operating System
- Hardware design experience

### 1.3 repo preparation

Download the source files:

`git clone https://github.com/cnrv/fpga-rocket-chip -b nexys4-demo`

` git submodule update --init --recursive`

There are several dirs in the repo:

- **constrains**, **verilog**, **firmware** - those are sources that we will use to build Vivado project and FPGA configration file, namely, mcs. Firmware contains the SDloader program that load elf image into DDR, functioning as a FSBL.
- **riscv-pk** - a modified verison of [riscv/riscv-pk](https://github.com/riscv/riscv-pk) repo. We plant the DTB into the bbl and slightly change the uart driver. Generally DTB should be located in the firmware, but for debugging convenience (it is time consuming to change the firmware and reburn it into FPGA), I put it just inside the bbl.

Besides, you also need to download the following repos:

- **cross compiler** - `git clone https://github.com/riscv/riscv-gnu-toolchain` 

- **linux kernel** -  ` git clone https://github.com/riscv/riscv-linux` 

- **busybox** that provides init and utils - `git clone https://github.com/mirror/busybox` 

## II. Hardware generation

**Vivado** version 2016.4; **Ubuntu** version 16 LTS

The tutorial should work well for most version of vivado.

Also, do not forget to install **vivado usb driver**, in case the Hardware Manager cannot recognize the board.

### 2.1 building the vivado project

#### 2.1.1 new project wizard

- Creat New Project 
- Add Sources - Add Directories - choose **/verilog**
- Add Constraints - Add Files - choose **/constrains/Board_Pin_Map.xdc**
- Default Part - Parts - choose **xc7a100tcsg324-1** , this is the chip that nexys4ddr holds.
- In the **Project Manager** window, right click **dut_inst - rocketTop (chip_top.v)** and set it as Top module

#### 2.1.2 Adding Peri IPs

**a) clocking wizard** 

- Component name - clk_wiz_0
- Clocking Options - Primitive - PLL
- Output Clocks - Output Clock - clk_out1 30.000; clk_out2 200.000
- Output Clocks - Enable Optional IO - check reset and locked
- Output Clocks - Reset Type - Active Low

**b) AXI crossbar**

- Component name - axi_crossbar_0
- Global: Num of Slave - 1; Num of Master - 3; Protocol - AXI4
- Global: Addr Width - 31; Data Width - 64; ID Width - 4
- Address: M00_AXI - 0x60000000 - 13; M01_AXI - 0x60010000 - 16; M00_AXI - 0x60020000 - 7

**c) AXI UART16550**

- [doc](https://www.xilinx.com/support/documentation/ip_documentation/axi_uart16550/v2_0/pg143-axi-uart16550.pdf)
- Component name - axi_uart16550_0
- AXI CLK Frequency - 30 MHz
- UART MOde - 16550

**d) AXI BRAM Controller**

- Component name - axi_bram_ctrl_0
- AXI protocol - AXI4; Data Width - 64; Memory Depth - 8192; ID Width - 4
- BRAM_INSTANCE - External; Num of BRAM interface - 1
- Enable ECC - no

**e) AXI Quad SPI**

- [doc](https://www.xilinx.com/support/documentation/ip_documentation/axi_quad_spi/v3_2/pg153-axi-quad-spi.pdf)
- Component name - axi_quad_api_0
- AXI Interface Options - uncheck all
- SPI options: Mode - standard; Transaction width - 8; Frequency Ration - 2x1; Num of Slaves - 1
- Check Enable Master Mode; Check Enable FIFO - FIFO Depth 16

**f) AXI Clock Converter**

- Component name - axi_clock_converter_0
- Protocol - AXI4; Read_Write Mode - read write; Addr Width - 32; Data Width - 64; ID Width - 4
- Clock Concersion Options - Asynchronous - Yes

#### 2.1.3 DDR controller 

- [doc](https://www.xilinx.com/support/documentation/ip_documentation/mig_7series/v4_2/ds176_7Series_MIS.pdf)
- IP Catalog - Memory Interface Generator
- MIG output options - Creat Design
- Check AXI4 Interface
- Pin Compatible FPGA - Select xc7a100ti-csd324
- Memory Selection - DDR2 SDRAM
- Options for Controller: Clock Period - 5000ps; PHY to Controller Clock Ration - 4:1
- Options for Controller: Memory Part - MT47H64M16HR-25E; Data Width - 16;
- Options for Controller: Num of Bank Machines - 4; Ordering - Normal
- AXI Parameter Options: Data Width - 64; Arbitration Scheme - RD_PRI_REG; ID Width: 4
- Memory Options for Controller: Input Clock Period - 5000ps (200MHz)
- Memory Options for Controller: Burst type - Sequential; Output Drive Strength - Full
- Memory Options for Controller: Controller Chip Select Pin - Enable; ODT - 50ohms
- Memory Options for Controller: Memory Address Mapping Selection - BANK/ROW/COLUMN
- System Clock - No Buffer; Reference Clock - Use System Clock
- System Reset Polarity - Active HIGH
- Debug Signal - off; check Internal Verf; IO Power Reduction - off; XADC - off
- Internal Termination Impedance - 50ohms
- Pin/Bank Selection Mode - Fixed Pin Out
- Pin Selection For Controller - Read XDC/UCF - select /constrain/DDR_Pin_Map.ucf - Validate
- Accept

### 2.2 generating MCS

- Generate BitStream
- This may take a long time: core i7-6700, 5min for synthesis, 7min for implementation
- Tools - Generate Memory Configuartion File
  - Format - MCS; Memory Part - s25fl128sxxxxxx0-spi; Interface - SPIx1
  - Do not forget to specify a filename and the output path
  - Check Load bitstream files - select  **your_project_dir/.runs/impl_1/.bit**

- Then you can find a ***.mcs** file under the path you specify in the Filename Option.
- Hardware Manager - add configuration memory device - s25fl128sxxxxxx0-spi
- Hardware Window - program configuration memory device
  - select your *.mcs file as configuration file
  - Apply

- Or if you wish you could use **Program Device** that sending bitstream to FPGA through USB directly, in which case, you need to program FPGA manually everytime you poweroff it.

## III. linux Image generation

### 3.1 generating DTB

TBA...

### 3.2 compiling Linux

### 3.3 build the image

## IV. Q&A, future map, known bugs

## V. Acknowledgement

Prof. Wei Song

lowrisc

bar-riscv



