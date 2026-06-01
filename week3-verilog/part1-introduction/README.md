# Part 1 — Introduction to Verilog

> Verilog does not describe what a program does step by step. It describes what a circuit IS. This is the fundamental mindset shift.

---

## Hardware Is Not Software

When you write Python or C, you write instructions that execute one after another. A CPU reads them in sequence.

When you write Verilog, you describe **concurrent hardware**. Every gate operates simultaneously. Every wire carries a value at every moment. There is no "next instruction" — it all happens in parallel, all the time.

```
Software thinking:          Hardware thinking:
  x = a + b                  Wire x is always connected to
  y = x * c                  the output of an adder
  z = y - 1                  whose inputs are wires a and b.
  (sequential)                (concurrent, always active)
```

This difference is why learning Verilog feels strange at first. You must think like an electrical engineer, not a programmer.

---

## What Is Synthesis?

**Synthesis** is the process of converting your Verilog description into a physical circuit.

```
You write:              Synthesis produces:         Implemented as:
┌──────────────┐       ┌──────────────────┐       ┌─────────────────┐
│  Verilog     │──────▶│ Gate-level Netlist│──────▶│ FPGA LUTs and   │
│  (.v files)  │       │ (AND, OR, FF...) │       │ routing switches │
└──────────────┘       └──────────────────┘       └─────────────────┘
```

Not all Verilog is synthesizable. Verilog can also be used for **simulation** — describing things that have no hardware equivalent (file I/O, delays measured in nanoseconds, print statements). Simulation constructs cannot be synthesized.

The distinction matters because synthesis tools will warn or error on non-synthesizable constructs in design code (though they are perfectly legal in testbenches).

---

## Synthesizable vs. Non-Synthesizable Verilog

| Construct | Synthesizable? | Notes |
|-----------|---------------|-------|
| `module`, `endmodule` | Yes | Basic structure |
| `wire`, `reg` | Yes | Signal declarations |
| `assign` | Yes | Combinational logic |
| `always @(posedge clk)` | Yes | Sequential logic (flip-flops) |
| `always @(*)` | Yes | Combinational logic |
| `if`, `case` | Yes | In always blocks |
| `for` (fixed bounds) | Yes | Unrolls into parallel hardware |
| `#10` (delay) | **No** | Simulation only |
| `$display`, `$monitor` | **No** | Simulation only |
| `initial` block | **No** (in general) | Simulation/testbench only |
| `while` loops | **No** (usually) | Cannot synthesize infinite loops |
| Real-valued math | **No** | No floating point in basic synthesis |

---

## The Module: Verilog's Basic Unit

Every piece of hardware in Verilog is a **module**. A module has:
- A name
- Ports (inputs and outputs to the outside world)
- Internal logic

Modules can instantiate other modules — this is how you build hierarchy.

```verilog
module my_module (
    input  wire  clk,      // clock input
    input  wire  rst,      // reset input
    input  wire  a,        // data input
    output wire  y         // data output
);

    // Internal logic goes here

endmodule
```

Think of a module like a chip. The ports are the pins on the package. Inside, the logic is implemented however you choose.

---

## The FPGA Design Flow (High Level)

Here is the full picture of what happens from Verilog to working hardware:

```
1. RTL Design (Register Transfer Level)
   You write Verilog describing your circuit at the
   behavioral/dataflow level.

2. Functional Simulation
   Before touching an FPGA, simulate your design.
   Run a testbench that applies inputs and checks outputs.
   Tool: iverilog + GTKWave (free), or Vivado simulator.

3. Synthesis
   The synthesizer converts Verilog to a gate-level netlist.
   Tool: Vivado (Xilinx), Quartus (Intel), Yosys (open source).

4. Implementation
   The implementation tool:
   a. Maps gates to FPGA primitives (LUTs, FFs, BRAMs)
   b. Places those primitives on the FPGA chip
   c. Routes the wires between them
   
5. Timing Analysis
   Verify that all paths between flip-flops complete
   within one clock period.

6. Bitstream Generation
   Produces a binary file that configures the FPGA.

7. Programming / Deployment
   Load the bitstream onto the FPGA via JTAG or SPI.
   Your hardware is now running on silicon.
```

---

## A Brief History of Verilog

**1984:** Phil Moorby at Gateway Design Automation (GDA) designs Verilog as a simulation language. The syntax draws from C and the simulation languages of the time.

**1990:** Cadence Design Systems acquires GDA. Cadence places Verilog in the public domain to build adoption.

**1995:** Verilog becomes IEEE Standard 1364-1995.

**2001:** Verilog 2001 adds significant improvements: ANSI-style port declarations, generate blocks, `always @(*)`, multi-dimensional arrays. Most FPGA code today uses Verilog 2001 or later.

**2005:** Verilog 2005 (minor revision).

**2009:** SystemVerilog (IEEE 1800) merges Verilog with SystemVerilog, adding:
- `logic` type (replaces most `wire`/`reg` confusion)
- Interfaces, packages, enums, structures
- Assertions, coverage
- Object-oriented testbench constructs

**Today:** SystemVerilog is the industry standard for verification. Synthesizable RTL often still uses Verilog 2001 style for compatibility, but `always_ff`, `always_comb`, and `logic` from SystemVerilog are widely used in new designs.

---

## Verilog vs. VHDL: Why Verilog?

| Feature | Verilog | VHDL |
|---------|---------|------|
| Syntax inspiration | C | Ada |
| Verbosity | Concise | Very verbose |
| Type system | Loosely typed | Strongly typed |
| Simulation speed | Faster | Slower |
| Industry preference | Stronger in US, Asia | Stronger in Europe, defense |
| FPGA tool support | Excellent | Excellent |
| Learning curve | Easier | Steeper |

Both work. Verilog (SystemVerilog) is the dominant language for FPGA development today.

---

## Next

In the next part, you will write your first Verilog modules — starting with the gate-level basics and working up to behavioral code.

**Next:** [Part 2 — Verilog Syntax & Basics](../part2-syntax-basics/README.md)
