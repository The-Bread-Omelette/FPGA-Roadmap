# Week 3 — Verilog HDL

> You now understand what hardware does. This week, you learn to describe it in code.

---

## Why Hardware Description Languages Exist

In the 1970s, designing a chip meant drawing schematics by hand — every gate, every wire. As circuits grew to millions of gates, this became impossible.

**Hardware Description Languages (HDLs)** were created to describe circuits textually — like writing code, but the "execution" is physical hardware. The compiler (called a **synthesizer**) reads your HDL and generates the actual gate-level netlist.

Two HDLs dominate the industry:

- **Verilog** — developed by Phil Moorby and Prabhu Goel at Gateway Design Automation in 1984. Acquired by Cadence in 1990, standardized as IEEE 1364 in 1995. Syntax influenced by C.
- **VHDL** — developed by the US Department of Defense in 1983. More verbose, strongly typed, influenced by Ada.

This course uses **Verilog** (specifically SystemVerilog for some modern constructs). It is the dominant language in industry for FPGA and ASIC design.

- You can practice writing verilog in [hdlbits](https://hdlbits.01xz.net/wiki/Problem_sets).

---

## Parts

| Part | Topic |
|------|-------|
| [Part 1 — Introduction to Verilog](./part1-introduction/README.md) | History, philosophy, what synthesis means |
| [Part 2 — Syntax & Basics](./part2-syntax-basics/README.md) | Modules, ports, wire, reg, always, assign |
| [Part 3 — Design Levels of Abstraction](./part3-design-levels/README.md) | Gate-level, dataflow, behavioral |
| [Part 4 — Simulation with iverilog & GTKWave](./part4-simulation/README.md) | Writing testbenches, running simulations |
| [Part 5 — Verilog Traps, Inference Rules & Everything the Synthesizer Does](./part5-verilog-traps-deep/README.md) | Operator Precedence, Signed Arithmetic, Latch Inference, Synthesis Rules |


---

Proceed to **[Part 1 — Introduction to Verilog](./part1-introduction/README.md)**

