# FPGA Development — Complete Learning Path

> From zero to building real hardware on FPGAs. No prior knowledge required.

---

## What Is This?

This is a course on FPGA development using Verilog HDL and Xilinx toolchains. It is designed like a textbook meets a workshop — every concept is explained from first principles, with enough depth that you could build and understand real hardware by the end.

If you have never touched digital electronics, hardware description languages, or FPGAs — start at Week 1 and work forward. By Week 8, you will understand how FPGAs work at the silicon level, how to write synthesizable hardware, how to deploy it, and how to build systems that bridge software and hardware.

---

## Course Map

| Week | Topic | Key Outcomes |
|------|-------|--------------|
| [Week 1](./week1-digital-electronics/README.md) | Digital Electronics Foundations | Number systems, logic gates, combinational circuits |
| [Week 2](./week2-sequential-circuits/) | Sequential Circuits & FSMs | Flip-flops, counters, state machines |
| [Week 3](./week3-verilog/) | Verilog HDL | Writing, simulating, and thinking in hardware |
| [Week 4](./week4-fpga-fundamentals/) | FPGA Architecture & Toolchain | How FPGAs work inside, installing Vivado |
| [Week 5](./week5-vivado-combinational/) | Vivado — From Code to Hardware | Synthesis, implementation, bitstream, timing |
| [Week 6](./week6-vitis-embedded/) | Vitis & Embedded Systems | PS/PL, AXI, bare-metal software on FPGA |
| [Week 7](./week7-advanced-hdl/) | Advanced HDL & Constraints | Pipelining, IP cores, timing closure |
| [Week 8](./week8-networking-interfaces/) | Interfaces & High-Speed IO | UART, SPI, I2C, Ethernet, DMA |

---

## How to Use This Course

Each week is a folder. Inside, you will find multiple parts — each part is a focused README covering one concept. Read them in order. Do not skip weeks.

- **Read** the concept explanation
- **Build** the exercise at the end of each part
- **Move on** only when the exercise compiles and makes sense

The exercises are designed to be done on any Xilinx/AMD FPGA board (Basys 3, Nexys A7, Arty, Zynq boards, etc). Most simulations require only a computer — no hardware needed until Week 5.

---

## Prerequisites

- A computer running Windows 10/11 or Ubuntu 20.04+
- ~100 GB free disk space (Vivado is large)

No programming experience required. No electronics background required.

---

## Tools You Will Install

| Tool | Purpose | When |
|------|---------|------|
| iverilog + GTKWave | Lightweight Verilog simulation | Week 3 |
| Xilinx Vivado | FPGA synthesis, implementation, bitstream | Week 4 |
| Vitis Unified IDE | Embedded software development | Week 6 |

---

Start here: **[Week 1 — Digital Electronics](./week1-digital-electronics/README.md)**
