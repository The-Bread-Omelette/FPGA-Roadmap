# Part 2 — Inside the FPGA: Architecture Deep Dive

> Understanding what is inside the chip explains why your design behaves the way it does.

---

## The Xilinx 7-Series Architecture

We use the Xilinx 7-series as our reference (Artix-7, Kintex-7, Virtex-7, and the FPGA fabric inside Zynq-7000). This architecture is well-documented, widely used in education and industry, and the one Vivado targets for most learning boards.

---

## Configurable Logic Block (CLB)

A **CLB** is the basic logic unit. In the 7-series, each CLB contains two **Slices**.

```
CLB
├── Slice L (Left)
└── Slice M (Right, has extra LUT RAM capability)
```

### Slice Structure

Each Slice contains:
- **4 six-input LUTs** (called A, B, C, D LUT)
- **8 flip-flops**
- **Wide multiplexers** (MUX8, MUX4)
- **Carry chain logic**

```
Slice:
┌─────────────────────────────────────────┐
│  LUT A ──┬──── FF A1                    │
│   6-in   │                              │
│          └──── FF A2                    │
│                                         │
│  LUT B ──┬──── FF B1                    │
│          └──── FF B2                    │
│                                         │
│  LUT C ──┬──── FF C1      Carry chain   │
│          └──── FF C2         ↑↓         │
│                                         │
│  LUT D ──┬──── FF D1                    │
│          └──── FF D2                    │
│                                         │
│  Wide MUX, Carry logic                  │
└─────────────────────────────────────────┘
```

### LUT in Detail

A 6-input LUT contains 64 SRAM bits. The inputs (I0-I5) address one of the 64 bits. Whatever is stored there is the output.

```
        I5 I4 I3 I2 I1 I0
         │  │  │  │  │  │
         └──┴──┴──┴──┴──┘
               │
               ▼ (6-bit address → 0 to 63)
        ┌──────────────┐
        │ 64 SRAM bits │
        │ (programmed  │
        │ by bitstream)│
        └──────────────┘
               │
               ▼
            Output O6
```

The same 64 bits can also be accessed as two independent 32-bit lookup tables (for 5-input functions), or as a 64×1 distributed RAM, or as a 32-bit shift register.

### Carry Chain

Each slice has a dedicated carry propagation circuit. This is not routing — it is a direct, short connection between adjacent slices specifically for arithmetic. This is why wide adders synthesize efficiently on FPGAs: the carries don't have to fight for general routing resources.

```
From slice below    → carry in
Carry logic         → fast carry propagation
To slice above      → carry out
```

A 32-bit adder on a 7-series uses the carry chain of 8 slices (4 LUTs per slice × 8 slices = 32 full adders). The carry chain delay is much faster than routing through LUTs.

---

## Block RAM (BRAM)

FPGAs contain dedicated **Block RAM** — large, fast, true-dual-port memory blocks embedded in the fabric.

### 7-Series BRAM Specs

- Each BRAM block: 36 Kb (36,864 bits)
- Can be configured as: 32K×1, 16K×2, 8K×4, 4K×9 (with parity), 2K×18, 1K×36
- Two independent ports (Port A and Port B) — each can read or write
- Both ports share the same memory, but operate independently with different clocks
- First-word fall-through or registered output

```
         Port A                     Port B
          ┌────┐                     ┌────┐
ADDRA ──▶│    │                     │    │◀── ADDRB
 DINA ──▶│    │  Shared 36Kb RAM    │    │◀── DINB
          │    │                     │    │
DOUTA ◀──│    │                     │    │──▶ DOUTB
 WEA  ──▶│    │                     │    │◀── WEB
 ENA  ──▶│    │                     │    │◀── ENB
CLKA  ──▶│    │                     │    │◀── CLKB
          └────┘                     └────┘
```

**Why BRAM matters:**

LUT-based distributed RAM is inefficient for large memories. If you need a 1024-byte buffer, implementing it in LUTs would use 1024 × 8 = 8192 LUT bits across hundreds of LUTs. One 36 Kb BRAM block holds 4096 bytes with zero LUT usage.

**Typical BRAM uses:**
- FIFOs between clock domains
- Lookup tables (sine wave ROM, coefficient tables)
- Packet buffers in networking
- Frame buffers in video
- Instruction memory for soft CPUs

### BRAM Primitives

```verilog
// Inferring BRAM in Verilog (synthesis infers RAMB36E1)
module simple_dpram #(
    parameter DEPTH = 1024,
    parameter WIDTH = 8
) (
    input  wire              clk,
    input  wire              we,
    input  wire [$clog2(DEPTH)-1:0] addr_w, addr_r,
    input  wire [WIDTH-1:0]  din,
    output reg  [WIDTH-1:0]  dout
);
    reg [WIDTH-1:0] mem [0:DEPTH-1];

    always @(posedge clk) begin
        if (we)
            mem[addr_w] <= din;
        dout <= mem[addr_r];   // registered output → infers BRAM
    end
endmodule
```

Synthesis tools recognize this pattern and map it to BRAM automatically.

[Learn more about block RAM](https://nandland.com/lesson-15-what-is-a-block-ram-bram/)

---

## DSP48 Slices

Digital Signal Processing requires multiplications — and lots of them. Multipliers implemented in LUTs consume enormous resources and are slow.

The 7-series DSP48E1 is a dedicated multiply-accumulate (MAC) unit:

```
DSP48E1:
               ACIN
                │
A (30-bit) ──▶ ┌─────┐
               │ PRE- │
B (18-bit) ──▶ │ADDER│──▶ A×B (48-bit) ──▶  ┌────────┐
               │      │                       │  POST  │──▶ P (48-bit)
C (48-bit) ──▶ └─────┘           PCIN ──────▶│  ADDER │
                                              └────────┘
                                                 │
                                               P (feedback)
```

**DSP48E1 can compute in one clock cycle:**
- A × B
- A × B + C
- A × B + P (accumulate)
- P + C (without multiply)
- And many more via cascading

**Real-world use:**
- FIR filter: each tap is one DSP (multiply + accumulate)
- FFT butterfly: two DSPs per stage
- Matrix multiply: one DSP per element
- PID controller: multiply setpoint by gain

An Artix-7 (XC7A100T) has 240 DSP48E1 slices. A Kintex-7 can have up to 2020.

---

## Clock Distribution — PLLs and MMCMs

The clock signal must arrive at every flip-flop at the same time (or with known, bounded skew). A dedicated clock network ensures this — it is separate from the general routing fabric and has very low, well-characterized skew.

### Global Clock Networks

The 7-series has up to 32 global clock networks. Signals on these networks can fan out to every flip-flop with very low skew (picoseconds).

### MMCM and PLL

**MMCM (Mixed-Mode Clock Manager)** and **PLL (Phase-Locked Loop)** are dedicated hardware blocks that can:
- Multiply the input clock frequency
- Divide the input clock frequency
- Generate multiple output clocks with different frequencies
- Shift the clock phase
- Eliminate clock jitter

```
External 100 MHz ──▶ MMCM ──▶ 100 MHz (system clock)
                          ──▶  50 MHz (half speed)
                          ──▶ 200 MHz (memory clock)
                          ──▶  25 MHz (Ethernet PHY clock)
                          ──▶  10 MHz (debug clock)
```

All five clocks are derived from one external oscillator, are synchronized, and have well-controlled phase relationships.

In Vivado, the MMCM is instantiated using the **Clocking Wizard IP** (IP catalog → clocking wizard → configure your frequencies → generate).

---

## I/O Blocks

I/O blocks (IOBs) sit at the periphery of the FPGA and connect to the physical package pins.

Each IOB contains:
- Input buffer
- Output buffer (with configurable drive strength and slew rate)
- Optional I/O flip-flops (very low latency for high-speed I/O)
- Optional differential signaling (LVDS)
- Programmable I/O standards: LVCMOS 1.8V, 2.5V, 3.3V; LVDS; HSTL; etc.

```
Package Pin ◀──▶ IOB ◀──▶ Internal Fabric
```

**I/O standards matter:** Match the I/O standard to the external device. Connecting a 5V signal to a 3.3V FPGA input will damage it. Connecting to an LVDS pair requires configuring the IOB for differential input.

In Vivado, I/O standards are set in the XDC constraints file:

```tcl
set_property PACKAGE_PIN U16  [get_ports {sw[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {sw[0]}]
```

---

## Hard IP Blocks

Modern FPGAs include "hard" (fixed silicon, not LUT-based) implementations of common functions:

### Ethernet MAC

The Zynq and larger 7-series devices include dedicated Ethernet MAC (Media Access Control) blocks that implement the data link layer of Ethernet without using any FPGA fabric.

### PCIe Endpoint

Hard PCIe blocks implement the PCIe physical and data link layers. This is used in data center FPGAs (Alveo) where the FPGA plugs into a PCIe slot on a server.

### SERDES (High-Speed Serial)

Transceiver blocks (GTX, GTH, GTZ) implement high-speed serial lanes at 1 Gbps to 32+ Gbps. Used for PCIe, 10/100G Ethernet, HDMI, DisplayPort, SATA, Aurora, and other protocols.

### ARM Processor (Zynq only)

In Zynq SoCs, dual ARM Cortex-A9 cores are fabricated as hard silicon alongside the FPGA fabric. This is discussed in detail in Week 6.

---

## Resource Counting Example

Let us look at the Xilinx Artix-7 XC7A100T (the FPGA on the Nexys A7 board):

| Resource | Count |
|----------|-------|
| Logic Cells | 101,440 |
| Slices | 15,850 |
| LUTs (6-input) | 63,400 |
| Flip-Flops | 126,800 |
| Block RAM (36Kb each) | 135 (4,860 Kb total) |
| DSP48E1 | 240 |
| MMCM | 6 |
| PLL | 6 |
| I/O Pins | 300 max |
| PCIe (hard block) | None |
| GTP Transceivers | 0 (Artix-7 uses GTP at lower speed) |

**What can you fit?**
- A simple RISC-V CPU: ~2,000-5,000 LUTs (fits easily)
- 1080p video processing pipeline: ~20,000-40,000 LUTs (feasible)
- A small neural network accelerator: depends on precision, but feasible
- A complete SSD controller: tight but possible
- Full Linux-capable CPU: need a Zynq (hard ARM cores)

---

## How Synthesis Maps to Resources

When you write Verilog and run synthesis, here is what gets mapped to what:

| Verilog Construct | FPGA Resource |
|-------------------|---------------|
| Combinational logic (assign, always @(*)) | LUTs |
| Sequential logic (always @(posedge clk)) | Flip-Flops |
| Multiplexers | LUTs or dedicated MUX resources |
| Small memories (<16 entries) | Distributed RAM (LUTs) |
| Large memories (>256 entries) | BRAM |
| Multipliers | DSP48E1 |
| Clock generation | MMCM/PLL |
| External signals | IOBs |

Vivado's utilization report (after synthesis or implementation) shows exactly how many of each resource your design uses. This is how you know if your design will fit on your target device.

---

## Video Resources

- [FPGA Architecture explained — Zynq Book](https://www.youtube.com/watch?v=bVFp2_ZM8xs)
- [LUTs and CLBs explained — FPGA fundamentals](https://www.youtube.com/watch?v=9VDPkbpH21M)
- [BRAM in Vivado](https://www.youtube.com/watch?v=SalKirqOc3k)
- [DSP48 deep dive](https://www.xilinx.com/support/documentation/user_guides/ug479_7Series_DSP48E1.pdf)

---

**Next:** [Part 3 — The Toolchain Ecosystem](../part3-toolchain-ecosystem/README.md)
