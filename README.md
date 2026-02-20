# Hardware-Software Co-Design of an ARMv4-T Pipelined CPU on NetFPGA

## Overview
This repository contains the RTL implementation and system-level integration of a multithreaded, pipelined ARMv4-T processor on the NetFPGA platform. The architecture demonstrates a complete Hardware/Software Co-design approach. By leveraging NetFPGA's memory-mapped I/O (MMIO) and generic register system, the host machine can dynamically program the CPU's Instruction Memory (IMEM), assert soft resets, and monitor hardware states (such as the Program Counter and current instructions) in real-time over the PCIe bus.

## System Architecture & FPGA Integration
The CPU is integrated into the NetFPGA's primary packet processing pipeline, the **User Data Path (UDP)**. The integration relies on a hierarchical wrapper design to isolate the CPU core logic from the complexities of the NetFPGA system buses.

1. **Host to FPGA:** The host PC executes a software script (`sw/armcpu`) that issues `regread`/`regwrite` commands. These commands travel over PCIe to the NetFPGA.
2. **Register Bus Routing:** The NetFPGA infrastructure routes these memory-mapped requests into the `user_data_path`, which forwards them to the custom `pc_wrapper`.
3. **Protocol Translation:** Inside the `pc_wrapper`, a `generic_regs` module translates the NetFPGA register bus protocol into simple parallel control signals (e.g., `sw_mem_addr`, `sw_mem_wdata`).
4. **Core Hijacking (In-System Programming):** When the software asserts a reset signal, the multiplexers within `pipelinepc.v` grant memory control to the host, allowing the software to flash machine code directly into the Block RAM (BRAM). Once the reset is deasserted, the CPU reclaims control and begins execution.

---

## Directory and File Description

### 📂 `include/` (System Configuration & Memory Mapping)
This directory contains the XML configuration files processed by the NetFPGA compilation scripts to automatically generate address decoders and C/Perl header files.
* **`project.xml`**: The top-level system allocation map. It declares the use of the custom CPU module within the UDP and assigns a base memory address to it, ensuring it does not conflict with other NetFPGA components (like Ethernet MACs or DMA queues).
* **`ids.xml`** *(Note: Functions as the CPU register map)*: Defines the specific Memory-Mapped I/O (MMIO) registers for the CPU. It distinguishes between **Software Registers** (`generic_software32`, written by the host PC to control reset and memory flashing) and **Hardware Registers** (`generic_hardware32`, written by the FPGA to report the PC and execution states back to the host).

### 📂 `src/` (Hardware RTL Source Code)
This directory contains the synthesizable Verilog modules.
* **`user_data_path.v`**: The overarching datapath module of the NetFPGA. It routes standard network packets (from Rx to Tx queues) and acts as the host for the CPU module, wiring the system register bus into the `pc_wrapper`.
* **`pc_wrapper.v`**: The critical bridge module. It serves two purposes:
  1. It instantiates the `generic_regs` module to parse incoming register read/write requests.
  2. It wraps the `pipelinepc` core, connecting the parsed 32-bit software signals to the CPU's ports while allowing standard NetFPGA network traffic (`in_data` to `out_data`) to pass through uninterrupted.
* **`pipelinepc.v`**: The core logic of the pipelined CPU. It includes the logic for multi-threading, the datapath stages (IF, ID, EX, MEM, WB), and the multiplexing logic required to switch Instruction Memory (IMEM) access between the internal Program Counter and the external software interface.
* **`components.v`**: A library file containing the essential sub-modules of the processor architecture, including the ALU, Register File, Control Unit, Branch Predictor, and Pipeline Registers.

### 📂 `sw/` (Host Software Interface)
This directory contains the user-space scripts used to interact with the FPGA.
* **`armcpu`**: A Perl-based Command Line Interface (CLI) executed on the host Linux environment. It provides high-level commands (e.g., `reset`, `pc`, `write`) to interface with the FPGA. It maps human-readable commands to the specific memory addresses defined in the `include/` directory, serving as the primary programmer and debugger for the CPU.