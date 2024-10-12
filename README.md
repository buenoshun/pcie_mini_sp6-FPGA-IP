# pcie_mini_sp6-FPGA-IP
An VHDL FPGA IP core to implement transaction layer logic for Xilinx Spartan6 PCIe hard-IP

To download: Click the zip file, on the next page click "view raw" to download it.

Originally hosted (in 2011) at:
https://opencores.org/projects/pcie_mini
Since the OpenCores.org admins have abandoned their website, and don't allow new users to register and download old projects (like this one), I, Istvan Nagy, the sole author, decided to re-release the project on GitHub. The new release is with a new, less restrictive license: "BSD zero clause", instead of the old LGPL. This will allow companies to use it in their product without restrictions or obligations.
The BSD 0-clause license: Copyright (C) 2024 by Istvan Nagy buenoshun@gmail.com 
Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted. THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE.

This is a PCI-express to Wishbone Bridge IP for Xilinx SPARTAN-6 FPGAs.
For newer devices like Virtex-7, Kintex/Virtex Ultrascale/+ see new project (WIP): https://opencores.org/projects/pcie_mini_axi4s_wb

Developer: Istvan Nagy, Bluechip Technology, 2011

Very often we want to make a peripheral card or a peripheral block on an x86 motherboard using an FPGA, but not necesserily want to spend a lot of time on developing common blocks (like a PCI-express interface), we want to focus on our own custom logic design instead and use completely implemented IP cores for the common blocks. The "mini" in the name doesn't refer to a minicard formfactor, but it just signifies that the core is small and is implemented as a single VHDL file. This is a simple implementation of a PCI-Express target to Wishbone master bridge.

The Xilinx Series-5/6 FPGAs have a built-in PCI-Express Endpoint Block, however it does not contain the packet encoding/decoding logic. This IP core (pcie_mini) implements the missing parts of the Xilinx core and also adds a Wishbone back-end interface. Strangely the Xilinx PCIE-EP handles packet encoding/decoding for configuration accesses, but not for memory accesses. This core interfaces the Xilinx PCIE-EP with its Transaction (TRN) interface. The Xilinx Series-7 FPGAs have a more complete PCIE-EP, but they also support using the TRN interface, but unfortunatelly they only support 64-bit/128bit parallel buses at the moment (November 2011), which would require a redesign of the pcie_mini_core. The user has to use the Xilinx Coregenerator to generate a PCIE-EP wrapper (xilinx source files) for the chosen target FPGA device. pcie_mini still needs the Xilinx PCIE Endpoint block and the GTP transceivers.

This core works with a fixed 256Mbytes memory window, only BAR0 is implemented. It was tested with 32/16/8-bit single memory read and write transactions. The data fields in the PCIe packets get endianess-swap by the core, to match the internal 32-bit bus layout with the 32bit variable view in the test software.
The core was tested on a x1 PCIe card (custom designed card having Spartan-6 LX45T FPGA on it) with nVidia chipset on the test motherboard, ISE 12.1, RW v1.4.9 (debug program) and Windows-XP (on Windows-7 accessing the core with a debug program without an installed driver is not possible). The wishbone addresses are byte addresses just like the PCI-express addressing. The Wishbone byte enables are derived from the PCIe packets.
The output address of the core contails the 2 LSBs as well. The TLP logic is capable of handling up to 1k bytes (256 DWORDs) payload data in a single PCIe transaction, and can handle only one request at a time. If a new request is arriving while processing the previous one (e.g. getting the data from a wishbone read), then the state machine will not process it immediately, or it will hang. So the user software has to wait for the previous read completion before issueing a new request. The multiple data DWORDs in a single TLP generate separate Wishbone transactions. Theoretical Performance: WishBone bus: 62.5MHz, 32bit, 2clk/access -> 125MBytes/sec. Consecutive single write test showed 15.625 MBytes/sec bandwidth, read performance is expected to be a lot lower due to the turnaround time of the read transactions. The maximum data throughput could be achieved when using the maximum data payload size (1kBytes), although it was not tested in lack of such test software. The core uses INTA virtual wire to signal interrupts. It uses a 100MHz ref clock. The The generated Xilinx core had to be edited manually to support 100MHz, as per Xilinx AR#33761.

Implementation tips:
-make use of the attached UCF file and edit it,
-Synthesis-KeepHierarchy=Yes,
-Translate-AllowUnmatched_XX_Constraints=Yes
-If there are timing errors we still might want to procees to be able to analyze the violation details. For this, we have to set the windows env.variable XIL_TIMING_ALLOW_IMPOSSIBLE to 1.
-In the synthesis properties, pack IOB registers = off, Set the "FSM Encoding Algorithm" to "user".
-To make the FPGA-chip to configure in 500ms, set the bitgen option "config rate" to 33 (MHz).

Resource utilization:
(The pci_mini with the xilinx PCIE-EP wrapper files together)
540 Slice registers,
779 Slice LUTs,
10 BRAMs (RAMB16BWER, 22kBytes total)
3 BUFGs
1 PLL

Files (browse CVS):
-xilinx_pcie2wb.vhd - this is the pcie_mini IP core
-blk_mem_gen_v4_1.vhd and .ngc - this is an internal BlockRam for the pcie_mini
-pcie*.vhd and gtpa1*.vhd - these are the wrapper files from the Xilinx CoreGenerator. The user might have to re-generate them in case of using a different FPGA device (these were generated for the XC6SLX45T-2FGG484) or different settings like SS-VID.
-pcie_mini_constraints.ucf - example constraints for the core to be included in your chip-level project. It has to be edited by the user.

Project Status:
All 3 revisions can be found inside the "\pcie_mini\trunk\main_sources" directory.
(v1.1) Tested and working with 32/16/8-bit single memory reads and writes, also fast consecutive single reads/writes.
(v1.2) interrupt fix by Stephen Battazzo
(v1.4) 64-bit read fix, support for unaligned 32-bit read, and custom BAR0 address space size. Compatibility for MSI and Legacy interrupts by Scott Cogan, FRIB

Test hardware:
Custom designed x1 PCIe data acquisition card having Spartan-6 LX45T FPGA on it, nVidia chipset on the test motherboard, ISE 12.1, RW v1.4.9 (debug program) and Windows-XP. The card has its own DDR2 memory and peripherals (200MSPS ADC, DSP, communications interface) that all access to the same DDR2 memory. The PCIe accesses were all accessing either internal control registers or the on-board memory.

To-do:
-Write a test program to generate bulk/burst read/write transactions where the TLP payload size >> 1 (eg 16Bytes or 1024Bytes). If someone could write such a program, then we could progress with the development to the high bandwidth bulk data transfer support.
