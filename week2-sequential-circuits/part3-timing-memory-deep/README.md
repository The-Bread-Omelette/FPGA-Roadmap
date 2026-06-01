# Part 3 — Timing Deep Dive: Metastability, Clock Quality, Reset, Memory Types

> An interviewer will ask "what is the MTBF of a synchronizer" or "what is the difference between clock skew and jitter." These are not trivia — they determine whether your hardware works reliably in production.

---

## Metastability — The Full Story

### What Physically Happens

A flip-flop is a bistable circuit — it has two stable states (Q=0 and Q=1) and one unstable equilibrium point between them.

```
Energy:
  ┌───────────────────────────┐
  │     ╲         ╱           │
  │      ╲       ╱            │   Unstable peak = metastable point
  │       ╲─────╱             │
  │   0    ╲   ╱    1         │
  └───────────────────────────┘
            Q voltage

When the FF resolves correctly: it slides down one side quickly.
When the FF goes metastable: it sits near the peak, resolving slowly.
```

The metastable state is unstable — thermal noise will eventually push it to 0 or 1. But "eventually" might be longer than one clock period.

### Mean Time Between Failures (MTBF)

The probability that a synchronizer fails (metastable state lasts longer than one clock period) follows an exponential distribution:

```
MTBF = (e^(T_r / τ)) / (f_clk · f_data · T_w)

Where:
  T_r    = resolution time available = T_period - t_setup - t_cq
  τ      = time constant of the FF's metastability resolution (device-specific, ~30-100 ps for modern FPGAs)
  f_clk  = clock frequency of destination domain
  f_data = frequency of data transitions (how often data can be metastable)
  T_w    = metastability window = t_setup + t_hold (time window where sampling is risky)
```

**Example calculation:**

Suppose:
- f_clk = 100 MHz → T_period = 10 ns
- t_setup = 0.5 ns, t_cq = 1.0 ns → T_r = 10 - 0.5 - 1.0 = 8.5 ns
- τ = 50 ps = 0.05 ns
- f_data = 10 MHz, T_w = 0.1 ns

```
MTBF = e^(8.5 / 0.05) / (100×10⁶ × 10×10⁶ × 0.1×10⁻⁹)
     = e^170 / (10⁻¹)
     = e^170 × 10

e^170 ≈ 10^73.8

MTBF ≈ 10^74 seconds ≈ 10^67 years
```

This is why the 2-FF synchronizer works in practice — the resolution time available is so large compared to τ that failure is essentially impossible.

**What if you use only 1 FF?** T_r = T_period - t_setup - t_cq - t_logic_after_FF. If there is 3 ns of logic after the synchronizer FF that must complete before the next FF's setup time, T_r shrinks dramatically, and MTBF can fall to seconds or milliseconds.

**Rule:** Never put combinational logic between a synchronizer FF and the next FF in the destination domain. The synchronizer FF's output must go directly to another FF's D input (or through trivial logic), not through a long combinational path.

---

## Clock Skew, Jitter, and Wander

These are three different phenomena that affect clock quality. Mixing them up in an interview is a red flag.

### Clock Skew

**Definition:** The spatial difference in clock arrival time at different points on the chip.

```
FF_A receives CLK at t=0.5 ns
FF_B receives CLK at t=0.8 ns

Skew = |0.8 - 0.5| = 0.3 ns
```

Skew is **deterministic and repeatable** — same every clock cycle.

**Positive skew** (source FF clock arrives before destination FF clock):
```
Setup check: data path must be < T_period + skew (skew helps setup)
Hold check:  data path must be > t_hold + skew (skew hurts hold)
```

**Negative skew** (source FF clock arrives after destination FF clock):
```
Setup check: data path must be < T_period - |skew| (hurts setup)
Hold check:  data path must be > t_hold - |skew| (helps hold)
```

**On FPGAs:** The global clock network minimizes skew (typically < 50-100 ps on a 7-series device). Using local routing for clocks creates large skew — never do this.

### Clock Jitter

**Definition:** Cycle-to-cycle variation in clock period. The clock does not arrive at exactly the same time every cycle — it fluctuates.

```
Expected CLK edge times: 0, 10, 20, 30, 40 ns (100 MHz)
Actual CLK edge times:   0, 10.1, 19.9, 30.2, 39.8 ns

Jitter: ±0.2 ns
```

Jitter is **random** — varies from cycle to cycle.

**Sources of jitter:**
- VDD/GND noise (power supply noise modulates clock path delay)
- Thermal noise in oscillator circuits
- PLL phase noise
- Electromagnetic interference

**Impact on timing:** Vivado's timing analysis includes a clock uncertainty budget that accounts for jitter. If your clock has 200 ps of jitter, Vivado subtracts 200 ps from your available timing margin.

```
create_clock -period 10.000 -name clk [get_ports clk]

Vivado automatically adds clock uncertainty based on MMCM/PLL specs.
You can also set it manually:
set_clock_uncertainty 0.200 [get_clocks clk]
```

### Wander

**Definition:** Very slow (< 10 Hz) variations in clock frequency. Relevant for telecommunications where clocks must be synchronized over long time periods. Not generally relevant for FPGA digital design but comes up in networking and clock recovery contexts.

### Duty Cycle

The fraction of the clock period that the clock is HIGH.

```
Ideal: 50% (5 ns high, 5 ns low for 100 MHz)

Real PLLs/MMCMs guarantee duty cycle within ±5%

Why it matters: some logic uses both rising and falling edges.
Double-data-rate (DDR) memory interfaces clock on both edges.
A 40/60 duty cycle means unequal setup margins for the two edges.
```

---

## Synchronous vs. Asynchronous Reset: The Complete Analysis

This is a very common interview question where shallow answers fail.

### Asynchronous Reset

```verilog
always @(posedge clk or posedge rst)
    if (rst) q <= 0;
    else     q <= d;
```

**Behavior:** Reset asserts and de-asserts independently of the clock.

**Advantages:**
- Works even if clock is stopped (during startup, when clock is not yet stable)
- Reset takes effect immediately — no clock cycle needed
- Required for some initialization sequences before the clock starts

**Disadvantages:**

1. **Reset assertion glitches can cause problems.** If rst glitches HIGH for just 1 ns (less than a clock period), it will still assert the FF. In a synchronous design, a 1 ns glitch on a data signal is harmless because the FF only samples at clock edges. Not so with async reset.

2. **Reset removal (de-assertion) metastability.** When rst goes LOW, the flip-flop must exit reset. If this happens near a clock edge, the FF can go metastable on its reset removal. One FF might exit reset one cycle before another, creating a temporary invalid state.

3. **Scan chain issues in ASIC DFT.** Asynchronous resets bypass the scan chain, complicating test patterns.

### Synchronous Reset

```verilog
always @(posedge clk)
    if (rst) q <= 0;
    else     q <= d;
```

**Behavior:** Reset only takes effect at the clock edge.

**Advantages:**
- Glitch-immune: a short rst pulse that doesn't span a full clock cycle is ignored
- No reset removal metastability — it is treated like any other data signal
- Scan-friendly for DFT

**Disadvantages:**
- Does not work if clock is stopped
- Requires one clock cycle to take effect (timing-critical if fast response is needed)
- Synthesis creates different hardware: the reset appears as an enable on the mux before the FF's D input, rather than using the FF's dedicated reset pin

**In practice on FPGAs:**

Xilinx 7-series FFs have a dedicated synchronous set/reset (SR) and asynchronous preset/clear (PRE/CLR) input. Using the dedicated pin is more efficient (no extra LUT needed for the mux). Vivado infers which pin to use based on your coding style:

- `always @(posedge clk or posedge rst)` → asynchronous CLR
- `always @(posedge clk)` with `if (rst)` → synchronous SR (uses LUT)

**The professional recommendation:** Use **asynchronous assert, synchronous de-assert** (covered in Week 7 CDC section). This gives you the startup reliability of async reset while eliminating the de-assertion metastability problem.

---

## Moore vs. Mealy Output Timing — Precisely

This distinction trips people up when they only have a superficial understanding.

### Moore: Output Registered

```
State register → Output logic → Output

CLK edge → state updates → (next clock) output appears
```

Moore outputs are **one clock cycle behind** the state transition. They are stable for an entire clock period (registered, no glitches).

### Mealy: Output Combinational

```
State register + Current inputs → Output logic → Output

CLK edge → state updates → inputs change → output changes IMMEDIATELY
```

Mealy outputs can change **within a clock cycle** when inputs change, without waiting for a clock edge. This makes Mealy machines faster (by one cycle) but potentially glitchy.

### The Glitch Problem in Mealy

If inputs glitch (even briefly), Mealy outputs glitch with them — because the output is a direct combinational function of inputs. This can cause problems if the output drives an asynchronous input somewhere.

**Solution:** Register the Mealy output:

```verilog
// Registered Mealy output — best of both worlds
always @(posedge clk)
    output_reg <= mealy_combinational_output;
```

This adds one cycle of latency but eliminates glitches. Common in professional designs.

---

## Memory Types: SRAM vs DRAM vs Flash vs EEPROM

An interviewer can go very deep here.

### SRAM (Static RAM)

**Cell structure:** 6 transistors (6T SRAM cell) — two cross-coupled inverters (4T) for storage + 2 access transistors.

```
         Word Line (WL)
              │
BL ──[T1]──┬──[T2]── /BL
           │  │
          [P] [N]  ← cross-coupled inverter 1
           │  │
          [N] [P]  ← cross-coupled inverter 2
           │  │
          GND GND
```

**Properties:**
- Retains data as long as power is applied (static — no refresh needed)
- Very fast access (< 1 ns for embedded SRAM, single ns for standalone)
- High power (all 6 transistors leaking)
- Large cell area (6 transistors vs DRAM's 1T1C)
- Used for: cache (L1/L2/L3), register files, FPGA block RAM, FPGA configuration SRAM

**FPGA relevance:** The LUT configuration bits and routing switch bits are stored in SRAM. Every CLB on an FPGA contains thousands of SRAM cells. This is why FPGAs must be reconfigured on every power cycle — SRAM is volatile.

### DRAM (Dynamic RAM)

**Cell structure:** 1 transistor + 1 capacitor (1T1C). The capacitor stores the bit.

```
      Word Line
          │
BL ──[T]──┴── [C]
              │
             GND
```

**Properties:**
- Compact (1T1C vs 6T for SRAM) → high density → cheap
- Capacitor leaks charge → **must be refreshed** every ~64 ms
- Slower than SRAM (10-100 ns)
- Destructive read: reading discharges the capacitor → must write back
- Used for: main system memory (DDR4, DDR5, LPDDR)

**Refresh:** A dedicated refresh controller reads every row periodically to restore charge. During refresh, the memory is unavailable — relevant for real-time systems.

**DDR (Double Data Rate):** Transfers data on both rising and falling clock edges, doubling bandwidth. DDR4 at 3200 MT/s means 3.2 billion transfers per second per pin, even though the actual clock is 1600 MHz.

### Flash Memory (NOR and NAND)

**Cell structure:** Floating-gate MOSFET. Charge trapped in the floating gate changes the transistor's threshold voltage.

```
         Control Gate
              │
         ┌────┴────┐ ← floating gate (electrically isolated)
         └────┬────┘
         ────── ── ──   ← thin oxide
Source ──[         ]── Drain
```

**NOR Flash:**
- Random access read (byte-addressable, like SRAM)
- Slow erase (sectors)
- Used for: code storage, boot ROMs, small data tables
- FPGA bitstream storage: NOR flash stores the bitstream that gets loaded on power-up

**NAND Flash:**
- No random access — page/block oriented
- Very high density, low cost per bit
- Limited erase cycles (~100K to 1M per block)
- Used for: SSDs, SD cards, USB drives, eMMC

**Flash on Zynq:** The PS can boot from QSPI NOR flash (holds FSBL + bitstream + application) or NAND flash or SD card (holds FAT filesystem with boot.bin).

### EEPROM

Electrically Erasable Programmable ROM. Byte-erasable (vs Flash which erases in blocks). Slower and more expensive than Flash but useful for storing configuration data that changes rarely (calibration values, serial numbers).

---

## Register File Architecture

A register file is an array of registers with multiple read and write ports. Every CPU has one.

```
32-register file with 2 read ports and 1 write port:

Read Port 1:  addr1[4:0] → data1[31:0]  (reads register #addr1)
Read Port 2:  addr2[4:0] → data2[31:0]  (reads register #addr2)
Write Port:   wr_addr[4:0], wr_data[31:0], wr_en → (writes register #wr_addr)

All reads are combinational. Write takes effect at clock edge.
```

**Implementation on FPGA:**

For small register files: use distributed RAM (LUTs) — supports synchronous write, combinational read.

For larger register files: use BRAM in simple dual-port or true dual-port mode.

**Read-during-write behavior:** What happens when you read and write the same address simultaneously?

- **Read-first** (also called "transparent"): read returns old value before write
- **Write-first**: read returns new (just-written) value
- **Don't care**: behavior is undefined, implementation-specific

FPGAs' BRAMs support all three modes — set in the Block Memory Generator IP. Incorrect mode selection is a subtle bug.

---

## Exercises

1. You have a 200 MHz design with a 2-FF synchronizer. τ = 40 ps, t_setup = 0.3 ns, t_cq = 0.8 ns, t_logic_after_sync = 1.5 ns, f_data = 50 MHz, T_w = 0.05 ns. Calculate the MTBF. Is this acceptable for a commercial product expected to run for 10 years?

2. Explain in detail why synchronous reset cannot be used in a system where the clock is stopped during some power-saving modes.

3. A DDR4-3200 memory interface has 64 data pins. What is the peak theoretical bandwidth in GB/s?

4. Draw the state diagram for a Mealy and Moore version of the same sequence detector ("101"). Show that the Mealy version requires one fewer state. Show the output timing difference in a timing diagram.

5. A NAND flash block has 1M erase cycles. An application writes 4 KB every second to the same block. How long before the block wears out? What technique would prevent premature wear-out?

6. In a register file with read-first behavior, a CPU reads register R5 in the same cycle it writes a new value to R5. What does the CPU read? Why might this cause a pipeline hazard in a pipelined processor?

---

**Next:** [Week 3 — Verilog HDL](../../week3-verilog/README.md)
