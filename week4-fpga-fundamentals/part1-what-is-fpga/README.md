# Part 1 — What Is an FPGA?

> Between software and silicon lies a remarkable device: hardware that can be programmed.

---

## The Problem FPGAs Solve

When you have a computing task, you face a fundamental trade-off:

```
                Fast, but               Flexible, but
                inflexible              slow
                    │                       │
                    ▼                       ▼
             ┌──────────┐           ┌──────────────┐
             │   ASIC   │           │     CPU      │
             │          │           │  (Software)  │
             │ Custom   │           │              │
             │ Silicon  │           │ General      │
             │          │           │ Purpose      │
             └──────────┘           └──────────────┘
                    
                    ↑ Between these two extremes ↑

                         ┌──────────┐
                         │   FPGA   │
                         │          │
                         │ Custom   │
                         │ hardware,│
                         │ but      │
                         │ reprog-  │
                         │ rammable │
                         └──────────┘
```

### CPUs (Software)

A CPU executes instructions sequentially. It is extremely flexible — you can run any software. But for tasks that require massive parallelism (video encoding, signal processing, encryption), a CPU is slow because instructions run one at a time.

### ASICs (Custom Silicon)

An ASIC (Application-Specific Integrated Circuit) is a chip designed and fabricated for exactly one purpose — like the Tensor Processing Units in Google data centers, or the H.264 encoder in your phone. ASICs are extremely fast and power-efficient, but cost millions of dollars to design and fabricate. Once made, you cannot change them.

### FPGAs

An FPGA (Field-Programmable Gate Array) sits between these. It is a chip that contains:
- Thousands to millions of configurable logic cells
- Programmable interconnects between them
- Specialized hard blocks (RAM, DSP, high-speed serial)

You configure it by loading a **bitstream** — a binary file that programs the logic cells and routing switches. You can reprogram an FPGA in seconds. Change your design, rerun synthesis, load the new bitstream.

---

## A Brief History of FPGAs

**1985:** Ross Freeman and Bernard Vonderschmitt co-found Xilinx and ship the **XC2064** — the world's first commercial FPGA. It contains 64 configurable logic blocks and 85,000 transistors. It could be programmed in the field (as opposed to during fabrication), giving FPGAs their name.

**1988:** Altera (later acquired by Intel) ships competing FPGA products. The two-horse race between Xilinx and Altera defines the industry for 30 years.

**1990s:** FPGAs grow from thousands to hundreds of thousands of logic cells. SRAM-based programming replaces the original fuse-based approach. Dedicated RAM blocks and multipliers appear on-chip.

**2000s:** Xilinx ships the Virtex-II Pro — the first FPGA with embedded PowerPC processor cores. This launches the era of "heterogeneous" FPGAs with both programmable logic and hard processor cores.

**2010:** Xilinx announces Zynq-7000 — a device with dual ARM Cortex-A9 processors tightly coupled to FPGA fabric via AXI buses. This changes FPGA development: now you can run Linux on the same chip as your custom hardware.

**2012:** Altera ships Cyclone V SoC with similar ARM+FPGA integration.

**2015:** Xilinx ships UltraScale+ with up to 4.5 million logic cells, HBM memory integration, and 100G networking support.

**2022:** AMD acquires Xilinx. The Versal series ships — combining ARM cores, AI engines (dedicated ML accelerators), and FPGA fabric in a single package.

**Today:** FPGAs range from tiny low-cost devices (Xilinx Spartan-7, Intel MAX 10) to massive compute accelerators used in data centers (Xilinx Alveo, Intel Stratix). The market is split between Xilinx/AMD and Intel/Altera, with Lattice Semiconductor serving low-power applications.

---

## Where Are FPGAs Used?

### Telecommunications

LTE, 5G base stations use FPGAs for baseband processing — the signal processing between the antenna and the network. FPGAs handle this because the standards change frequently (5G evolved multiple times in 2 years), and ASICs would be obsolete before they ship.

### Data Centers

Microsoft's Project Brainwave uses FPGAs in Azure data centers to accelerate machine learning inference. Every Bing search query passes through an FPGA for ranking. Xilinx Alveo cards are used for network packet processing (SmartNICs).

### High-Frequency Trading

Financial firms use FPGAs to implement trading logic with single-digit microsecond latency. Software running on a CPU has latency measured in microseconds to milliseconds — too slow.

### Aerospace and Defense

Radar signal processing, electronic warfare, satellite communication. FPGAs are used here because they are rad-hardened (resistant to cosmic radiation) and can be reconfigured in flight if the mission changes.

### Medical Devices

MRI machines, ultrasound imaging, patient monitoring. FPGAs process image data in real time. Medical devices require long lifetimes and often need updates without hardware replacement.

### Prototyping ASICs

Before taping out a billion-dollar chip, engineers prototype it on FPGAs. A 100,000-LUT FPGA can hold a simplified model of a complex processor. Bugs found on FPGA cost days; bugs found after tape-out cost millions.

### Video and Image Processing

4K video encoding, machine vision, real-time object detection. FPGAs implement the DSP algorithms (FIR filters, FFTs, convolutions) that are too compute-intensive for CPUs but too specialized for dedicated chips.

### Automotive

ADAS (Advanced Driver Assistance Systems) use FPGAs for sensor fusion — combining data from radar, lidar, and cameras. Xilinx's Zynq is used in Tesla Autopilot and many other autonomous driving systems.

---

## How Does an FPGA Work?

At its core, an FPGA contains three things:

1. **Configurable Logic Blocks (CLBs)** — implement any logic function
2. **Programmable Interconnect** — route signals between blocks
3. **I/O Blocks** — connect to external pins

```
┌──────────────────────────────────────────────────────────┐
│                         FPGA                             │
│                                                          │
│  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐          │
│  │CLB │══│CLB │══│CLB │══│CLB │══│CLB │══│CLB │          │
│  └────┘  └────┘  └────┘  └────┘  └────┘  └────┘          │
│    ║       ║       ║       ║       ║       ║             │
│  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐          │
│  │CLB │══│CLB │══│CLB │══│CLB │══│CLB │══│CLB │  I/O     │
│  └────┘  └────┘  └────┘  └────┘  └────┘  └────┘  Blocks  │
│    ║       ║       ║       ║       ║       ║             │
│  ...    ...    ...    ...    ...    ...                  │
│                                                          │
│  ══ = Programmable routing switches                      │
└──────────────────────────────────────────────────────────┘
```

When you load a bitstream, it configures:
- The logic function implemented in each CLB
- Which routing switches are open or closed (connecting or disconnecting wires)

### The Look-Up Table (LUT)

The core of a CLB is a **Look-Up Table (LUT)**.

An N-input LUT is essentially a truth table stored in SRAM. It can implement **any** Boolean function of N inputs:

```
4-input LUT:

Inputs A, B, C, D (4 bits → 16 possible combinations)

┌───┬───┬───┬───┬────────┐
│ A │ B │ C │ D │ Output │
├───┼───┼───┼───┼────────┤
│ 0 │ 0 │ 0 │ 0 │   0    │  ← SRAM bit [0]
│ 0 │ 0 │ 0 │ 1 │   1    │  ← SRAM bit [1]
│ 0 │ 0 │ 1 │ 0 │   0    │  ← SRAM bit [2]
│ ...                    │
│ 1 │ 1 │ 1 │ 1 │   1    │  ← SRAM bit [15]
└───┴───┴───┴───┴────────┘

The 16 SRAM bits can be set to any pattern,
implementing any 4-input logic function.
```

The inputs address the LUT like a 16-entry memory. The output is whatever is stored at that address. Change the stored pattern → change the implemented function.

Modern Xilinx FPGAs use **6-input LUTs** (64 SRAM bits). Altera/Intel uses similar architectures.

### Routing

Between logic blocks, there is a sea of wires and programmable switches. The routing network can connect any output to any input, at the cost of delay.

When Vivado "places and routes" your design, it:
1. Decides which LUTs implement which logic functions (placement)
2. Decides which routing switches to close to connect them (routing)

The routing delay is often the limiting factor for clock speed — a long route between two flip-flops means the clock must be slower to give the signal time to arrive.

---

## FPGA vs. Microcontroller vs. ASIC — When to Use What

| Criteria | Microcontroller | FPGA | ASIC |
|----------|----------------|------|------|
| Development time | Hours–days | Days–weeks | Months–years |
| Nonrecurring cost | Low | Low–medium | Very high (millions) |
| Per-unit cost | Low (high volume) | Medium–high | Very low (high volume) |
| Performance | Moderate | High | Very high |
| Power consumption | Low–moderate | Moderate | Low |
| Flexibility | High (software) | High (reprogrammable) | None |
| Parallel processing | Poor | Excellent | Excellent |
| Time to market | Fast | Fast | Slow |
| Typical volumes | Any | Low–medium | Very high |

**Choose FPGA when:**
- You need hardware-level performance but can't afford/justify ASIC cost
- The algorithm or standard may change
- You are prototyping an ASIC
- You need custom I/O interfaces
- You need massive parallelism (DSP, image processing)
- You need low latency (sub-microsecond response)

---

## Major FPGA Vendors and Families

### Xilinx / AMD

| Family | Category | Notable Features |
|--------|----------|-----------------|
| Spartan-7 | Low-cost | Good for learning, cheap boards |
| Artix-7 | Cost-optimized | Good balance of cost and performance |
| Kintex-7 | Performance | High LUT count, DSP, high-speed serial |
| Virtex-7 | High-end | Maximum performance |
| Zynq-7000 | SoC | Dual ARM + 7-series FPGA |
| Zynq UltraScale+ | Advanced SoC | Quad ARM + UltraScale FPGA |
| Versal | AI/ML | ARM + AI Engine + FPGA |

### Intel (formerly Altera)

| Family | Category |
|--------|----------|
| MAX 10 | Low-cost, non-volatile |
| Cyclone V/10 | Cost-optimized |
| Arria 10 | Mid-range performance |
| Stratix 10 | High performance |
| Agilex | Latest generation |

### Lattice Semiconductor

Focuses on low-power, small FPGAs. Popular in mobile, industrial, and automotive.

### Microchip (formerly Microsemi)

Radiation-hardened and space-grade FPGAs (ProASIC, SmartFusion).

---

## Popular FPGA Development Boards

| Board | FPGA | Good For |
|-------|------|----------|
| Basys 3 (Digilent) | Xilinx Artix-7 | Learning, combinational circuits |
| Nexys A7 (Digilent) | Xilinx Artix-7 | More I/O, audio, Ethernet |
| Arty A7 (Digilent) | Xilinx Artix-7 | MicroBlaze soft CPU |
| PYNQ-Z2 (TUL) | Xilinx Zynq-7020 | Python + FPGA, ML |
| ZedBoard | Xilinx Zynq-7020 | SoC development |
| DE10-Nano (Terasic) | Intel Cyclone V SoC | HPS + FPGA |
| iCEBreaker | Lattice iCE40 | Open-source tools, learning |
| Tang Nano (Sipeed) | Gowin | Very cheap (~$5), for basics |

---

**Next:** [Part 2 — Inside the FPGA Architecture](../part2-architecture-deep-dive/README.md)
