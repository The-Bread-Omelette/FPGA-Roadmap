# Part 3 — Levels of Abstraction in Verilog

> The same circuit can be described in multiple ways. Choosing the right level of abstraction is a skill.

---

## The Three Levels

### 1. Gate-Level (Structural)

You instantiate primitive gates explicitly. This is closest to the actual circuit.

```verilog
// Full adder — gate level
module full_adder_gate (
    input  wire a, b, cin,
    output wire sum, cout
);
    wire w1, w2, w3;

    xor g1 (w1,  a,   b  );    // w1 = a XOR b
    xor g2 (sum, w1,  cin);    // sum = w1 XOR cin
    and g3 (w2,  a,   b  );    // w2 = a AND b
    and g4 (w3,  w1,  cin);    // w3 = w1 AND cin
    or  g5 (cout, w2, w3 );    // cout = w2 OR w3

endmodule
```

**When to use:** When you need precise control of the gate structure, or when working with a gate-level netlist from synthesis. Rarely written by hand for new designs.

---

### 2. Dataflow (RTL — Register Transfer Level)

You use `assign` statements with operators. The synthesizer decides how to implement the operators as gates. This is the most common style for combinational logic.

```verilog
// Full adder — dataflow
module full_adder_df (
    input  wire a, b, cin,
    output wire sum, cout
);
    assign sum  = a ^ b ^ cin;
    assign cout = (a & b) | (b & cin) | (a & cin);
endmodule
```

More compact, more readable, same result. The synthesizer maps `^`, `&`, `|` to XOR, AND, OR gates automatically.

---

### 3. Behavioral (Algorithmic)

You describe what the circuit *does* using `always` blocks, if/case statements, and arithmetic. The synthesizer infers the circuit from the behavior.

```verilog
// Full adder — behavioral
module full_adder_beh (
    input  wire a, b, cin,
    output reg  sum, cout
);
    always @(*) begin
        {cout, sum} = a + b + cin;
        // The {cout, sum} concatenation captures
        // the 2-bit result of the 1-bit addition
    end
endmodule
```

Even more compact. The synthesizer infers adder logic automatically.

**When to use:** Behavioral is preferred for complex arithmetic, state machines, and control logic. It produces more readable code and lets the synthesizer optimize.

---

## Which Level Should You Use?

| Level | When to Use |
|-------|-------------|
| Gate-level | Rarely (post-synthesis verification, manual optimization) |
| Dataflow (assign) | Simple combinational logic, MUXes, assignments |
| Behavioral (always) | FSMs, complex combinational logic, all sequential logic |

Most professional RTL code mixes dataflow and behavioral. You will rarely write gate-level by hand.

---

## The Importance of Thinking RTL

**RTL** (Register Transfer Level) is the mental model you use when writing synthesizable Verilog:

- Think about what happens at each **clock edge**
- Think about which signals are **registers** (flip-flops) and which are **wires**
- Think about how data **flows** from register to register through combinational logic

```
        Registers           Combinational Logic         Registers
        ┌──────┐           ┌──────────────────┐         ┌──────┐
D[7:0] ─┤  FF  ├─── Q[7:0]─┤  adder, mux,     ├── D ──▶┤  FF  ├─ Q
        └──────┘           │  comparator...   │         └──────┘
           ↑               └──────────────────┘            ↑
          CLK                                             CLK
          
          ← one clock cycle →
```

The synthesizer takes this RTL description and maps it to LUTs (for combinational logic) and flip-flops — the actual resources inside an FPGA.


> Please complete [hdlbits](https://hdlbits.01xz.net/wiki/Problem_sets) till the heading verification. Without practice, you can not consider yourself knowing verilog.
     


**Next:** [Part 4 — Simulation with iverilog & GTKWave](../part4-simulation/README.md)
