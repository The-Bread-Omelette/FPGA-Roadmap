# Part 5 — FPGA Internals Deep: Configuration, Routing, IOB, Reconfiguration

> How does a bitstream actually configure an FPGA? What are the five routing tiers? What is partial reconfiguration? These are the questions that test whether you understand FPGAs or just use them.

---

## How SRAM-Based FPGA Configuration Works

Every FPGA contains millions of SRAM bits. These bits are the "programming" — they control every LUT function, every routing switch, every IOB standard. Understanding this reveals everything about FPGA power, configuration time, and non-volatility.

### The Configuration Memory

```
The entire FPGA is a large SRAM array organized as a bitstream:

Bitstream structure (Xilinx 7-series):
  ┌─────────────────────────────────────────┐
  │  Sync Word (0xAA995566) — identifies    │
  │  a valid 7-series bitstream             │
  ├─────────────────────────────────────────┤
  │  Configuration Registers                │
  │  (COR0, COR1 — startup timing options)  │
  ├─────────────────────────────────────────┤
  │  Frame Data Packets                     │
  │  Each frame = 101 32-bit words          │
  │  (a vertical column of configuration)   │
  ├─────────────────────────────────────────┤
  │  CRC check word                         │
  └─────────────────────────────────────────┘
```

The FPGA has a dedicated **configuration logic block** separate from the main fabric. On power-up or on receipt of a configuration command, it reads frames from the bitstream and writes them into the SRAM array sequentially.

### Configuration Modes

| Mode | Interface | Where Used |
|------|-----------|-----------|
| JTAG | 4-wire JTAG | During development (Vivado Hardware Manager) |
| Master SPI | Up to ×8 SPI | Production: external SPI flash |
| Master BPI | Parallel NOR flash | High-speed, large bitstreams |
| Slave Serial | Single bit, external clock | Processor-managed config |
| Slave SelectMAP | 8/16/32-bit parallel | High-speed, FPGA-to-FPGA |

**JTAG** is what Vivado uses during development. The JTAG interface is always available regardless of the boot mode pins — it is hardwired and cannot be disabled by normal means (an important security consideration).

**On Zynq:** The PS can program the PL directly using the PCAP (Processor Configuration Access Port). This is what the FSBL does — it reads the bitstream from flash and writes it through PCAP to the PL. This makes Zynq self-contained: no external programmer needed.

### Configuration Time

A typical Artix-7 100T bitstream is ~17 MB. Over a ×4 SPI flash at 66 MHz:

```
17 MB × 8 bits/byte / (4 bits/cycle × 66 MHz) = 515 ms
```

Over JTAG at 10 MHz (during development):

```
17 MB × 8 / 10 MHz = 13.6 seconds
```

This is why iterating on large FPGA designs during hardware bringup can be time-consuming. Partial reconfiguration addresses this.

### Volatile vs. Non-Volatile

**SRAM-based FPGAs** (virtually all modern Xilinx/Intel FPGAs) lose configuration on power-off. They need an external non-volatile memory (SPI flash) to reload the bitstream on startup.

**Flash-based FPGAs** (Microchip ProASIC, Lattice MachXO) store the configuration in on-chip flash cells — they are instantly ready after power-on and retain configuration without external memory. Trade-off: flash cells are slower to program, have limited erase cycles, and the process is harder to integrate with advanced nodes.

**Anti-fuse FPGAs** (Microsemi, obsolete Actel): configured once at the factory using high-voltage anti-fuse programming. Cannot be erased. Used in radiation-hardened (satellite, military) applications where SRAM-based configuration could be corrupted by cosmic rays.

---

## FPGA Routing Architecture — Five Tiers

This is what makes FPGA routing different from an ASIC. The routing is hierarchical.

### Tier 1: Local Interconnect (within a CLB)

Within a single CLB/slice, outputs from LUTs can feed directly into other LUTs in the same slice without entering the general routing. This is the fastest interconnect — essentially zero extra delay.

```
LUT_A → directly feeds LUT_B within same slice (cascade MUX)
```

### Tier 2: Direct Connections (adjacent CLBs)

Each CLB has direct connections to its four immediate neighbors (N, S, E, W). Used by the place-and-route tool when it can arrange logic to use these fast connections.

### Tier 3: Double/Hex Lines (local routing)

Short wires that span 2 or 6 CLBs in a row or column, with programmable switches at each CLB they pass through.

```
CLB─S─CLB─S─CLB─S─CLB─S─CLB─S─CLB
    (S = programmable switch)
```

### Tier 4: Long Lines (global routing)

Wires that span the entire chip width or height. Used for:
- High-fanout signals (global reset, enable signals)
- When placement requires long connections

Long lines have higher delay but are needed when logic is far apart.

### Tier 5: Global Clock Networks

Completely separate network dedicated to clocks. The 7-series has 12 global clock buffers (BUFG) and 12 regional clock buffers (BUFR). These drive a dedicated, balanced H-tree network that reaches every flip-flop with minimal and equal delay (< 50-100 ps skew across the device).

**Rule:** Clocks must always use BUFG or BUFR. Never route a clock on normal routing — you will get enormous skew and Vivado will warn you.

```verilog
// Explicitly instantiate a global clock buffer (usually auto-inferred)
BUFG u_bufg (
    .I (clk_from_pin),
    .O (clk_global)     // this signal is on the global clock network
);
```

### Why Routing Delay Dominates

For a modern 7-series FPGA at 100-200 MHz:
- LUT propagation delay: ~0.1–0.4 ns
- Local routing (double lines): ~0.3–0.6 ns
- Long routing: 1.0–3.0 ns

A design with poor placement will use long routing on critical paths. This is why a design that "should" run at 300 MHz might only achieve 150 MHz if placement is bad — the logic is fast but the wires are long.

---

## IDELAY and ODELAY — Input/Output Delay Primitives

For high-speed interfaces (DDR, LVDS, source-synchronous buses), you need fine-grained control over the delay of input and output signals.

**IDELAY (Input Delay):** Adds a programmable delay to an input signal before it enters the fabric.

```
Pin → IBUF → IDELAY → FF or fabric
              ↑
         0 to 31 taps × ~78 ps per tap (200 MHz reference clock)
         = 0 to ~2.4 ns adjustable delay
```

**Use case:** Source-synchronous interfaces where the data and clock arrive at slightly different times due to PCB trace length differences. Tune IDELAY to align the data edge to the center of the clock cycle.

```verilog
(* IODELAY_GROUP = "my_group" *)
IDELAYE2 #(
    .IDELAY_TYPE  ("FIXED"),
    .IDELAY_VALUE (10),           // 10 taps × 78 ps = 780 ps delay
    .REFCLK_FREQUENCY (200.0)
) u_idelay (
    .C       (idelay_ctrl_clk),
    .IDATAIN (data_from_ibuf),
    .DATAOUT (data_to_fabric),
    .LD      (1'b0),
    .CE      (1'b0),
    .INC     (1'b0),
    .CINVCTRL(1'b0),
    .CNTVALUEIN(5'd0),
    .CNTVALUEOUT()
);
```

**IDELAYCTRL** is a control block that must be instantiated in each bank using IDELAY. It generates the tap calibration from a 200 MHz reference clock.

---

## IOB Flip-Flops — Why They Exist

Each I/O block has its own flip-flops (one for input, one for output). Using these instead of the fabric flip-flops reduces I/O path delay.

```
Without IOB FF:
  Pin → IBUF → fabric routing → FF (fabric) → data
  Delay: pad → FF = IBUF delay + routing

With IOB FF:
  Pin → IBUF → FF (in IOB, right next to pad) → data
  Delay: pad → FF = IBUF delay only (routing delay eliminated)
```

This matters for timing-critical high-speed interfaces where even a few hundred picoseconds of routing delay can cause setup violations.

**How to infer IOB flip-flops:**

```verilog
// Synthesizer may infer IOB FF automatically if the FF is directly at a port
// You can force it with IOB attribute:
(* IOB = "TRUE" *) reg data_in_reg;

always @(posedge clk)
    data_in_reg <= data_in_pin;  // this FF goes into the IOB
```

Or in XDC:
```tcl
set_property IOB TRUE [get_cells data_in_reg]
```

---

## Partial Reconfiguration

**Partial Reconfiguration (PR)** allows you to reconfigure a portion of the FPGA fabric while the rest continues operating. The non-reconfigured regions are unaffected.

### Use Cases

- **Software-defined radio**: Swap modulation/demodulation algorithms at runtime without stopping the system
- **Adaptive computing**: Change the processing algorithm based on input data type
- **FPGA time-sharing**: Run different accelerators in the same physical region, like virtual machines for hardware
- **Field firmware updates**: Update a peripheral's implementation without resetting the entire system

### How It Works

```
Physical FPGA:
┌────────────────────────────────────┐
│        Static Region               │
│  (never reconfigured, always runs) │
│                                    │
│  ┌──────────────┐                  │
│  │  Reconfigurable│                │
│  │   Region 1   │   ← Only this   │
│  │  (slot)      │     is reloaded │
│  └──────────────┘                  │
│                                    │
│  ┌──────────────┐                  │
│  │  Reconfigurable│                │
│  │   Region 2   │                  │
│  └──────────────┘                  │
└────────────────────────────────────┘
```

Vivado's PR flow:
1. Define reconfigurable partitions (Pblocks)
2. Synthesize the static region and all module variants independently
3. Implement all variants — the static region is locked, only the reconfigurable slot changes
4. Generate a full bitstream (initial load) and partial bitstreams (for each variant)

Partial bitstreams are small (~1/10th the full size) and can be loaded in milliseconds.

---

## SelectIO — I/O Standards in Detail

The I/O pins on an FPGA are not simple TTL inputs/outputs. They support dozens of electrical standards, each suited to different applications.

### Single-Ended Standards

| Standard | Voltage | Drive | Typical Use |
|----------|---------|-------|-------------|
| LVCMOS33 | 3.3V | 4-24 mA | LEDs, switches, low-speed peripherals |
| LVCMOS25 | 2.5V | 4-16 mA | Some sensors, interfaces |
| LVCMOS18 | 1.8V | 4-12 mA | Modern peripherals |
| LVCMOS12 | 1.2V | 4-8 mA | Very low power |
| LVTTL | 3.3V (TTL thresholds) | 4-24 mA | Legacy compatibility |

### Differential Standards

Differential signaling sends two complementary signals. The receiver detects the difference between them, which is immune to common-mode noise (both signals are affected equally by noise, the difference is unaffected).

```
Signal P: ───────╱─────╲──────
Signal N: ───────╲─────╱──────

Differential voltage = P - N

Noise on both lines equally:
Signal P: ──────╱──────╲──── (+ noise)
Signal N: ──────╲──────╱──── (+ same noise)
P - N: unaffected ← noise cancels
```

| Standard | Voltage Swing | Speed | Use |
|----------|--------------|-------|-----|
| LVDS | ±350 mV differential | Up to 1 Gbps | General high-speed |
| TMDS | Variable | Up to 3.4 Gbps per lane | HDMI, DVI |
| HSTL | 1.5V | 800 MHz | DDR memory interfaces |
| SSTL | 1.2-1.8V | DDR3/4 speeds | DDR memory |

```verilog
// Differential input (LVDS)
IBUFDS u_ibufds (
    .I  (data_p),   // positive pin
    .IB (data_n),   // negative pin
    .O  (data_single_ended)
);

// Differential output
OBUFDS u_obufds (
    .I  (data_out),
    .O  (data_p),
    .OB (data_n)
);
```

---

## Exercises

1. An Artix-7 A100T bitstream is loaded over JTAG at 15 MHz. Estimate the programming time. Now estimate it over ×4 SPI at 100 MHz.

2. Your design uses a 200 MHz clock. The global clock network has 80 ps skew. A path between two FFs has a combinational delay of 4.7 ns. The FFs have t_setup = 0.4 ns and t_hold = 0.1 ns. Does it meet timing? Show the setup and hold checks.

3. Explain why routing delay increases non-linearly as a design gets more full (e.g., >80% LUT utilization).

4. A DDR4 memory interface uses SSTL12 differential. Why is differential signaling used instead of LVCMOS12? What would happen if you used single-ended at 3.2 GT/s?

5. What is the purpose of the BUFGMUX primitive, and when would you use it instead of a plain BUFG?

6. Describe a real application where partial reconfiguration provides a concrete business or technical advantage over two separate FPGAs.

---

**Week 5:** [Vivado — From Verilog to Bitstream](../../week5-vivado-combinational/README.md)
