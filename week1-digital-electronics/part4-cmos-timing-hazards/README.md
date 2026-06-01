# Part 4 — CMOS Internals, Propagation Delay, Hazards & Advanced Gates

> An interviewer who asks "why does CMOS consume power only when switching" is testing whether you actually understand the physics or just memorized a fact. This section gives you the physics.

---

## CMOS Power Dissipation — The Complete Picture

CMOS (Complementary MOS) is famous for near-zero static power. But power has three distinct sources, and each matters in practice.

### 1. Dynamic Power (Switching Power)

Every time a gate switches, it charges or discharges the load capacitance.

```
P_dynamic = α · C_L · V_DD² · f

Where:
  α    = activity factor (fraction of clock cycles where output switches, 0 to 1)
  C_L  = load capacitance (gate + wire + fan-out inputs)
  V_DD = supply voltage
  f    = clock frequency
```

**Why V²?** Charging a capacitor C from 0 to V stores energy ½CV². Discharging wastes the same. Total per switching cycle = CV². Multiply by frequency and activity factor.

**Implication for FPGAs:** Running at 200 MHz uses 2× the dynamic power of 100 MHz. Reducing V_DD from 1.8V to 1.0V reduces power by 3.24×. This is why modern FPGAs use 1.0V core voltage.

### 2. Short-Circuit Power (Shoot-Through)

During a transition, there is a brief moment when both PMOS and NMOS are partially on simultaneously. A direct current path exists from VDD to GND.

```
Slow input transition:

VDD
 │
[PMOS]  ← partially on during transition
 │
Out
 │
[NMOS]  ← partially on during transition
 │
GND

Both on simultaneously → current flows directly from VDD to GND
```

Well-designed circuits minimize this by ensuring fast transitions. Short-circuit power is typically 10-20% of dynamic power.

### 3. Leakage Power (Static Power)

MOSFETs are not perfect switches. Even when "off," a small subthreshold current flows. At advanced nodes (28nm, 16nm, 7nm), this becomes significant.

```
Types of leakage:
  Subthreshold leakage: current flows when Vgs < Vth
  Gate leakage:         tunneling through ultra-thin gate oxide
  Junction leakage:     reverse-biased p-n junctions
```

At 65nm and below, leakage power can exceed dynamic power at low activity. This is why modern SoCs use power gating — physically cutting VDD to idle blocks.

**On FPGAs:** Leakage is why a powered but unconfigured FPGA still consumes current. The device is full of transistors all leaking slightly.

---

## CMOS Transmission Gate

A transmission gate passes a signal (nearly) unchanged in either direction. It solves the threshold drop problem of a single MOSFET pass transistor.

```
Circuit:
        A ───┤ NMOS ├─── B
             ├──────┤
        A ───┤ PMOS ├─── B
    
Control: C  ──────────────── NMOS gate
Control: C' (inverted) ───── PMOS gate

When C=1: NMOS on, PMOS on → signal passes from A to B (or B to A)
When C=0: both off → high impedance, signal blocked
```

**Why both NMOS and PMOS?** An NMOS pass transistor cannot pass a strong '1' (the output is V_DD - V_th, losing voltage). A PMOS cannot pass a strong '0'. Together they pass rail-to-rail.

Used in: MUX implementations, analog switches, clock gating cells, D latches.

---

## Tristate Buffers and Buses

A tristate buffer has three possible output states: 0, 1, and **Z** (high impedance — effectively disconnected).

```
         ┌──────┐
In ─────▶│      │
         │  Tri ├─── Out
OE ─────▶│      │    OE=1: Out = In (buffer drives the line)
         └──────┘     OE=0: Out = Z  (buffer disconnects)
```

**Why this matters:** Multiple devices can share one wire (bus) if only one drives at a time. The others are in high-Z.

```
   Device A                Device B                Device C
  ┌──────┐               ┌──────┐               ┌──────┐
  │  Tri ├── OE_A        │  Tri ├── OE_B        │  Tri ├── OE_C
  └──────┘    │          └──────┘    │          └──────┘    │
      │        │              │        │              │        │
      └────────┴──────────────┴────────┴──────────────┘
                                BUS
      
  Only one OE is asserted at a time. Bus arbiter ensures this.
```

**On FPGAs:** Tristate is supported on I/O pins (IOBUF primitive) and was once used inside the fabric (in older devices). Modern FPGA fabric avoids internal tristate — use MUXes instead. Vivado will warn if you infer internal tristate logic.

### IOBUF — Bidirectional I/O

```verilog
// Bidirectional I/O pin (e.g., I2C SDA, memory data bus)
IOBUF u_iobuf (
    .IO  (io_pin),      // connects to FPGA package pin
    .I   (data_out),    // data to drive onto pin
    .O   (data_in),     // data read from pin
    .T   (tristate_en)  // 1 = high-Z (input mode), 0 = drive (output mode)
);
```

---

## Open-Drain / Open-Collector

An open-drain output can only pull low. It cannot drive high — it relies on an external pull-up resistor.

```
                VDD
                 │
              [R_pullup]   ← external, typically 4.7kΩ or 10kΩ
                 │
Out ─────────────┤
                 │
              [NMOS]  ← in the driving device
                 │
                GND

Driver on:  NMOS on → Out pulled to GND (0)
Driver off: NMOS off → Out pulled to VDD via R (1)
```

**Why use open-drain?**

1. **Wired-AND**: Multiple open-drain outputs on the same wire. If ANY device drives low, the bus is low. Only HIGH when ALL devices release.
   ```
   I2C uses this: any device can pull SDA/SCL low for arbitration or clock stretching.
   ```

2. **Voltage level shifting**: A 3.3V FPGA open-drain output can drive a 5V bus by using a 5V pull-up resistor.

3. **Overcurrent protection**: No shoot-through between VDD and GND.

---

## Propagation Delay vs. Contamination Delay

Two different delay specifications for every gate:

### Propagation Delay (t_pd)

The time from when the input reaches 50% of its final value to when the output reaches 50% of its final value. This is the **worst-case** delay.

```
Input:   ─────────┐
                  └─────
         
Output:  ──────────────┐          (inverter example)
                       └─

t_pd = time from input crossing 50% to output crossing 50%
```

Used for: **Setup time analysis** — you need to know the *maximum* delay to ensure data arrives before the clock edge.

### Contamination Delay (t_cd)

The time from when the input *starts to change* to when the output *starts to change*. This is the **best-case** (minimum) delay.

```
A gate may start changing its output before the input fully settles.
t_cd < t_pd always.
```

Used for: **Hold time analysis** — you need to know the *minimum* delay to ensure data doesn't change too quickly after the clock edge.

**The Setup/Hold timing window:**

```
Previous FF     Combinational Logic      Current FF
   Q ─────────► [path: t_cd to t_pd] ──────► D
             ↑                              ↑
           CLK edge                       CLK edge (one period later)
   
   Hold check:  t_cd_min > t_hold
   Setup check: t_pd_max < T_period - t_setup
```

This is why **hold violations cannot be fixed by slowing the clock** — hold depends on minimum delay, not clock period. Fix hold by adding buffers (inserting minimum delay) on the path.

---

## Combinational Hazards (Glitches)

A **hazard** is a momentary incorrect output during a transition. The output should be stable at 1 or 0, but briefly glitches to the wrong value.

### Static-1 Hazard

Output should remain 1, but briefly glitches to 0.

**Example:** Two-gate AND-OR circuit

```
Y = AB + AC

When A=1, B=1, C=1 (Y should be 1):

If B→0 (while C=1):
  AB goes 0 immediately
  AC goes 0 after gate delay
  
  Brief moment: AB=0, AC=0 → Y=0 (glitch!)
  
  After: AB=0, AC=1 → Y=1 (correct final state)
```

```
Timing:
B:  ─────────┐
             └────────
             
AB: ─────────┐
             └────────
             
AC: ──────────────────   (lagging due to gate delay)
             │
             └──┐   ← small delay here
                └────
                
Y:  ─────────┐  ┌────   ← GLITCH during this window
             └──┘
```

### K-Map Detection of Hazards

A static-1 hazard occurs when two adjacent 1-cells in the K-map are covered by **separate** groups (not overlapping).

```
   Y = A'B + AB'

K-map:        A
         0    1
    B=0 │ 0 │ 1 │
    B=1 │ 1 │ 0 │

Groups: top-right (A=1, B=0) = AB'  and  bottom-left (A=0, B=1) = A'B

These groups don't overlap. Transition A=1,B=0 → A=0,B=1 (Y stays 1):
  During transition, both might briefly be 0 → Y=0 glitch.

... For XOR there's no simple fix without extra logic.
```

**Fix:** Add a redundant group that overlaps both, covering the transition. The extra term keeps Y=1 during the transition.

### Why Hazards Matter (and When They Don't)

- **In synchronous design**: Hazards on combinational outputs generally don't matter — the FF only samples at the clock edge, by which time all glitches have settled (this is why setup time analysis uses propagation delay, not contamination delay for outputs).
- **In asynchronous logic or direct hardware control** (e.g., controlling a power switch): Glitches are dangerous and must be eliminated.
- **When feeding asynchronous inputs** (e.g., interrupt lines, enable pins of external ICs): Glitches can cause false triggers.

---

## Fan-In and Fan-Out

### Fan-Out

The maximum number of inputs a single gate output can reliably drive.

In CMOS, fan-out is limited by:
1. **DC loading**: Each input draws leakage current. High fan-out → accumulated leakage → output voltage degrades
2. **AC loading**: Each input adds capacitance. High fan-out → slower transitions → increased t_pd

In FPGAs, the routing network handles fan-out, but very high fan-out (>50) creates timing problems because the router must use long routes to reach all destinations.

### Fan-In

The maximum number of inputs to a single gate. Beyond ~4-6 inputs, CMOS gates become slow (series stacked transistors increase resistance).

```
4-input NAND (CMOS):
VDD
 │
[P][P][P][P]   ← 4 PMOS in parallel (fast)
 │
Out
 │
[N]            ← 4 NMOS in SERIES (slow — 4× resistance)
[N]
[N]
[N]
 │
GND
```

This is why logic synthesis breaks large fan-in functions into trees of smaller gates, and why FPGAs use 6-input LUTs (instead of a single 32-input gate).

---

## Propagation Delay in CMOS: Rise vs. Fall

PMOS and NMOS transistors have different mobilities:
- Electron mobility (NMOS): ~450 cm²/V·s
- Hole mobility (PMOS): ~200 cm²/V·s

PMOS is roughly 2× slower than NMOS at the same size. To get equal rise and fall times, PMOS transistors are made ~2× wider than NMOS. This is why CMOS gates are not perfectly symmetric.

**Implication:** t_pHL (high-to-low transition) is typically faster than t_pLH (low-to-high) in an inverter with equal-sized transistors.

In FPGAs: The LUT and routing delays are characterized separately for rising and falling edges. Vivado's timing analysis considers both — the reported slack uses the worst of the two.

---

## The Buffer / Inverter Chain Trick

For high fan-out signals (like clock enables, global resets), the standard technique is to drive the signal through a chain of progressively larger buffers:

```
Source (weak) ──► [INV x1] ──► [INV x4] ──► [INV x16] ──► many loads
                  ↑
                  Each stage is 4× bigger

Why: Optimal buffer ratio for minimum total delay is e ≈ 2.7 (use 3-4 in practice)
```

In Vivado, this is called **fanout optimization** and is done automatically. The `MAX_FANOUT` attribute tells synthesis when to start replicating drivers.

---

## Exercises

1. Calculate the dynamic power of a 100 MHz design with 10,000 LUTs, activity factor 0.25, load capacitance 20 fF per LUT, and V_DD = 1.0V.

2. A gate has t_pd = 500 ps and t_cd = 200 ps. Another gate feeds into it with t_pd = 300 ps and t_cd = 100 ps. What are the cumulative t_pd and t_cd of the two-gate path?

3. Explain why hold violations cannot be fixed by reducing the clock frequency.

4. Design a 4-to-1 MUX using only transmission gates. How many transistors does your implementation use?

5. The K-map for Y = A'B'C + A'BC + ABC' + AB'C' has a static-1 hazard on a specific transition. Identify the transition and the redundant term that eliminates it.

6. An I2C bus has 2 devices, both trying to transmit simultaneously. Explain the arbitration mechanism using the open-drain bus property.

---

**Next:** Continue to [Week 2 — Sequential Circuits](../../week2-sequential-circuits/README.md)
