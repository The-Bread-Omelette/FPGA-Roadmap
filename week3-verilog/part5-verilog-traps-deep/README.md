# Part 5 — Verilog Traps, Inference Rules & Everything the Synthesizer Does

> The difference between a junior and senior Verilog engineer is not knowing the syntax — it is knowing exactly what hardware every line of code will infer, and why certain patterns silently produce wrong results.

---

## Operator Precedence — The Complete Table

Operator precedence errors are silent bugs. Memorize this or you will spend hours on them.

```
Highest precedence (evaluated first):
  1.  { }  {{ }}          Concatenation, replication
  2.  !  ~  &  ~&  |  ~|  ^  ~^    Unary (logical NOT, bitwise NOT, reductions)
  3.  **                  Power
  4.  *  /  %             Multiply, divide, modulo
  5.  +  -                Binary add, subtract
  6.  <<  >>  <<<  >>>    Shift (logical and arithmetic)
  7.  <  <=  >  >=        Relational
  8.  ==  !=  ===  !==    Equality
  9.  &                   Bitwise AND
  10. ^  ~^               Bitwise XOR, XNOR
  11. |                   Bitwise OR
  12. &&                  Logical AND
  13. ||                  Logical OR
  14. ?:                  Ternary (conditional)
Lowest precedence (evaluated last)
```

**Classic trap:**

```verilog
// Intent: (a | b) & c
assign y = a | b & c;    // WRONG: evaluates as a | (b & c)  — & binds tighter

// Correct:
assign y = (a | b) & c;
```

```verilog
// Intent: negate the XOR result
assign y = ~a ^ b;       // WRONG: evaluates as (~a) ^ b

// Correct (XNOR):
assign y = ~(a ^ b);
assign y = a ~^ b;       // or use XNOR operator
```

**The equality trap:**

```verilog
// === vs ==
// == compares values, treats x/z as unknown → result can be x
// === (case equality) compares including x and z — result always 0 or 1

reg [3:0] a = 4'bxxxx;

a == 4'bxxxx   // result is x (unknown!)
a === 4'bxxxx  // result is 1 (exact match including x bits)

// Use === only in simulation (testbenches). In synthesis, x doesn't exist.
```

---

## Signed Arithmetic — The $signed() Trap

Verilog variables are **unsigned by default**. This causes silent wrong answers when you do arithmetic that should be signed.

```verilog
reg [7:0] a = 8'd200;  // = -56 if interpreted as signed
reg [7:0] b = 8'd10;

// Unsigned comparison (default):
if (a > b) ...   // TRUE: 200 > 10 ✓ (but if you wanted signed comparison...)

// If a is meant to be -56:
if ($signed(a) > $signed(b)) ...   // FALSE: -56 < 10 ✓ (correct for signed)
```

```verilog
// Arithmetic right shift: >>> preserves sign bit
reg signed [7:0] x = -8;  // 11111000
x >>> 1;   // 11111100 = -4  (sign bit preserved) ✓
x >> 1;    // 01111100 = 124 (zero-filled)         ✗ for signed

// Or use $signed:
$signed(x) >>> 1;  // correct arithmetic right shift
```

**Declaring signed:**

```verilog
reg signed [7:0] s_val;   // always treated as signed
wire signed [15:0] s_wire;

// Or cast temporarily:
$signed(unsigned_reg)    // treat as signed for this expression
$unsigned(signed_reg)    // treat as unsigned for this expression
```

---

## Width Arithmetic — The Hidden Overflow Trap

Verilog extends the width of an expression based on the widest operand. But the width of the *context* (left-hand side) is not considered during computation — only for truncation/assignment.

```verilog
reg [7:0]  a = 8'hFF;   // 255
reg [7:0]  b = 8'h01;   // 1
reg [7:0]  c;
reg [8:0]  d;

c = a + b;  // 255 + 1 = 256 = 9'h100
            // 8-bit context: c = 8'h00  (carry LOST!) ← BUG

d = a + b;  // 255 + 1 = 256
            // 9-bit context: d = 9'h100 = 256 ✓

// FIX: zero-extend before adding
d = {1'b0, a} + {1'b0, b};   // forces 9-bit computation

// Or rely on the LHS width for arithmetic:
d = a + b;  // OK because d is 9 bits — Verilog uses max(9, 8, 8)=9 bits
```

**The carry/overflow fix:**

```verilog
// Capture carry-out of 8-bit add:
wire [7:0] sum;
wire       carry;

{carry, sum} = a + b;   // 9-bit result split into carry + 8-bit sum
```

---

## Latch Inference: Every Case That Creates One

An unintended latch is one of the most common FPGA bugs. Here are all the cases.

### Case 1: Incomplete if-else

```verilog
// INFERS LATCH for y when sel=0
always @(*) begin
    if (sel) y = a;
    // no else → y must HOLD its old value when sel=0 → latch
end

// FIX: assign default first
always @(*) begin
    y = 1'b0;        // default
    if (sel) y = a;  // override
end
```

### Case 2: Incomplete case

```verilog
// INFERS LATCH for y when sel is 2'b11
always @(*) begin
    case (sel)
        2'b00: y = a;
        2'b01: y = b;
        2'b10: y = c;
        // 2'b11 not handled → latch!
    endcase
end

// FIX: add default
always @(*) begin
    case (sel)
        2'b00: y = a;
        2'b01: y = b;
        2'b10: y = c;
        default: y = 1'b0;  // catch-all
    endcase
end
```

### Case 3: Not all outputs assigned in all branches

```verilog
// INFERS LATCH for y_b when sel=1
always @(*) begin
    if (sel) begin
        y_a = a;
        y_b = b;
    end else begin
        y_a = c;
        // y_b not assigned → latch for y_b!
    end
end

// FIX: assign all outputs a default at the top
always @(*) begin
    y_a = 1'b0;
    y_b = 1'b0;
    if (sel) begin
        y_a = a;
        y_b = b;
    end else begin
        y_a = c;
    end
end
```

### How to Detect Latches

Synthesis will warn: `"WARNING: [Synth 8-327] inferring latch for variable 'y'"`

Also, in Vivado schematic view, latches appear as `LDCE` primitives (level-sensitive D latch with clock enable). If you see these and didn't intend them, you have a bug.

**Exception:** Sometimes latches are intentional (e.g., level-sensitive enable on a datapath, clock gating cells). But in synchronous FPGA design, they are almost always a mistake.

---

## What Synthesizes to What — Complete Reference

| Verilog Pattern | Synthesized Hardware |
|----------------|---------------------|
| `assign y = a & b` | AND gate (LUT) |
| `always @(*)` with full case/if | Combinational LUT(s) |
| `always @(*)` with incomplete case/if | Latch (LDCE) |
| `always @(posedge clk)` | Flip-flop (FDRE) |
| `always @(posedge clk or posedge rst)` | Flip-flop with async reset (FDCE) |
| `always @(posedge clk)` with synchronous `if (rst)` | Flip-flop with sync reset (FDRE with CE from mux) |
| `assign y = {a, b}` | Wiring (no gate) |
| `assign y = a[3:0]` | Wiring (bit select, no gate) |
| `assign y = a + b` | Adder (carry chain + LUTs) |
| `assign y = a * b` | Multiplier (DSP48 or LUTs, depending on width) |
| `ram[addr] <= data` (in `posedge clk` block) | BRAM or distributed RAM write port |
| `data <= ram[addr]` (in `posedge clk` block) | BRAM read port (registered output) |
| `data = ram[addr]` (in `always @(*)` block) | Distributed RAM read (combinational) |
| `reg [N-1:0] shift; always @(posedge clk) shift <= {shift[N-2:0], in}` | SRL (shift register LUT) or FF chain |
| `for` loop with fixed bounds in `always @(*)` | Parallel combinational hardware (unrolled) |
| `for` loop with fixed bounds in `always @(posedge clk)` | Parallel flip-flops (unrolled) |
| `initial` block | Ignored in synthesis (simulation only) |
| `#10` delay | Ignored in synthesis |
| `$display` | Ignored in synthesis |

---

## parameter vs. localparam vs. defparam

```verilog
// parameter: can be overridden at instantiation time
module adder #(parameter WIDTH = 8) (input [WIDTH-1:0] a, b, output [WIDTH:0] sum);
    assign sum = a + b;
endmodule

// Override at instantiation:
adder #(.WIDTH(16)) my_adder(.a(x), .b(y), .sum(z));

// localparam: internal constant, CANNOT be overridden
// Use for internal values derived from parameters:
module adder #(parameter WIDTH = 8) (...);
    localparam HALF = WIDTH / 2;  // internal, not overridable
endmodule

// defparam: override a parameter by hierarchical path (DEPRECATED)
// Never use in new designs — breaks when hierarchy changes
defparam top.u_adder.WIDTH = 16;  // DON'T DO THIS
```

---

## Compiler Directives

```verilog
// `define — macro definition (text substitution, global scope)
`define CLK_PERIOD 10
always #(`CLK_PERIOD/2) clk = ~clk;

// `ifdef / `ifndef / `elsif / `endif — conditional compilation
`define SIMULATION
`ifdef SIMULATION
    // testbench-only code
    $display("Entering state %d", state);
`endif

// `include — include another file
`include "definitions.vh"   // .vh = Verilog header (convention)

// `timescale — sets simulation time unit and precision
`timescale 1ns / 1ps    // 1 ns unit, 1 ps precision
// Must appear before any module that uses delays
// #10 means 10 × 1 ns = 10 ns

// `default_nettype — controls implicit net declarations
`default_nettype none   // ALWAYS USE THIS
// Without it: undeclared identifiers become implicit wires
// With `default_nettype none: typos cause compile errors, not silent bugs

// `undef — undefine a macro
`undef CLK_PERIOD
```

**`default_nettype none is critical.** Without it:

```verilog
module top(input clk, output reg q);
    always @(posedge clk) q <= dta;  // typo: 'dta' instead of 'data'
    // Without `default_nettype none: 'dta' is implicitly declared as a 1-bit wire
    // tied to 0. No error, but wrong behavior.
    // With `default_nettype none: compile error "dta is not declared"
endmodule
```

---

## The `specify` Block (Timing Models)

Used to model propagation delays in simulation (for gate-level netlists), not in RTL synthesis. But you will encounter it in vendor simulation models.

```verilog
module my_gate (input a, b, output y);
    specify
        (a => y) = 1.2;      // a to y path: 1.2 ns
        (b => y) = 0.8;      // b to y path: 0.8 ns
        specparam tpd = 1.0;
    endspecify
    
    assign y = a & b;
endmodule
```

When you run post-synthesis simulation, the tool replaces your RTL modules with gate-level models that include `specify` blocks. This is why post-synthesis simulation is slower but more accurate.

---

## race conditions in simulation

Two always blocks that both trigger at the same simulation time step can produce different results depending on which runs first — a non-deterministic simulation.

```verilog
// RACE CONDITION:
always @(posedge clk) a = b;  // blocking
always @(posedge clk) b = a;  // blocking

// Who runs first? If block 1 first: a=b, then b=a (b gets new a, not original)
// If block 2 first: b=a, then a=b (a gets new b, not original)
// Result is non-deterministic between simulators!

// FIX: use non-blocking assignments
always @(posedge clk) a <= b;  // all RHS evaluated first
always @(posedge clk) b <= a;  // then all updates applied
// Result: a and b swap deterministically, regardless of block order
```

This is the deep reason why the non-blocking assignment rule exists — it prevents simulation race conditions from creating simulation/synthesis mismatches.

---

## `force` and `release` (Simulation Only)

Used in testbenches to override a signal's value, bypassing the normal driver:

```verilog
// Force a signal to a specific value regardless of what drives it
force dut.internal_wire = 1'b1;

// Release the force (signal returns to its driven value)
release dut.internal_wire;

// force on a reg:
force dut.state = 3'b010;   // override FSM state for testing
#100;
release dut.state;
```

Useful for: injecting faults, testing error recovery paths, overriding initialization sequences.

---

## Exercises

1. Predict the output of this simulation. Explain each step:
   ```verilog
   reg [3:0] a = 4'b1111;
   reg [3:0] b = 4'b0001;
   reg [3:0] c;
   reg [4:0] d;
   initial begin
       c = a + b;
       d = a + b;
       $display("c=%b d=%b", c, d);
   end
   ```

2. Write a 4-state FSM in Verilog. Introduce a deliberate latch bug, run synthesis, find the warning, and fix it.

3. What does this synthesize to? Is it a latch, FF, or combinational?
   ```verilog
   always @(en or data)
       if (en) q = data;
   ```

4. Explain why `===` cannot be used in synthesizable code.

5. Write a parameterized N-bit shift register that compiles without warnings under `` `default_nettype none ``. Include a testbench.

6. A colleague writes: `assign carry_out = (a + b) > 8'hFF;` where a and b are 8-bit regs. Will carry_out ever be 1? Explain why or why not, and fix the code if needed.

---

**Week 4:** [FPGA Architecture & Toolchain](../../week4-fpga-fundamentals/README.md)
