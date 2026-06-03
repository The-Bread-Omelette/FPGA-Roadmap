# Part 2 — Verilog Syntax & Basics

> Verilog has a small core vocabulary. Master these, and you can describe any digital circuit. This module is gonna take some time!

---

## Data Types

### wire

A `wire` models a physical wire — a connection. Its value is driven by whatever is connected to it. You cannot "assign to" a wire inside an `always` block.

```verilog
wire a, b, c;           // single-bit wires
wire [7:0] data_bus;    // 8-bit wire (bus)
wire [31:0] addr;       // 32-bit wire
```

### reg

A `reg` can hold a value — it models a storage element. Despite the name, a `reg` does **not** always synthesize to a flip-flop. A `reg` inside an `always @(*)` block synthesizes to combinational logic. A `reg` inside an `always @(posedge clk)` block synthesizes to a flip-flop.

```verilog
reg q;                  // 1-bit register
reg [7:0] counter;      // 8-bit register
reg [3:0] state;        // 4-bit state register
```

### integer, real (simulation only)

`integer` is a 32-bit signed variable used in testbenches. `real` is a floating-point variable. Neither synthesizes meaningfully.

### Logic Values

Verilog uses a 4-value logic system:

| Value | Meaning |
|-------|---------|
| `0` | Logic zero (GND) |
| `1` | Logic one (VDD) |
| `x` | Unknown / uninitialized |
| `z` | High impedance (floating, undriven) |

In simulation, `x` propagates through logic — if you use an uninitialized value, the error is visible. In real hardware, there is no `x` or `z` — everything is 0 or 1.

---

## Number Literals

```verilog
// Format: <width>'<base><value>
// Base: b (binary), o (octal), d (decimal), h (hex)

4'b1010     // 4-bit binary: 1010
8'hFF       // 8-bit hex: 11111111 = 255
16'd1024    // 16-bit decimal: 1024
8'b0000_1111  // underscores for readability

// Without width specifier (32-bit by default):
100         // decimal 100

// Special values:
4'bx        // 4 bits all unknown
4'bz        // 4 bits all high-Z
```

---

## Module Structure

```verilog
module module_name #(
    // Parameters (optional)
    parameter DATA_WIDTH = 8,
    parameter DEPTH      = 16
) (
    // Port list
    input  wire              clk,
    input  wire              rst_n,     // active-low reset
    input  wire [DATA_WIDTH-1:0] data_in,
    output reg  [DATA_WIDTH-1:0] data_out,
    output wire                  valid
);

    // Internal signals
    wire [DATA_WIDTH-1:0] internal_wire;
    reg  [DATA_WIDTH-1:0] internal_reg;

    // Logic...

endmodule
```

---

## assign — Continuous Assignment

`assign` creates a combinational connection. The right-hand side is continuously evaluated; whenever any input changes, the output updates immediately (in simulation, after propagation delay; in hardware, as fast as the gates allow).

```verilog
// Basic gate operations
assign y   = a & b;          // AND
assign y   = a | b;          // OR
assign y   = a ^ b;          // XOR
assign y   = ~a;             // NOT
assign y   = ~(a & b);       // NAND
assign y   = ~(a | b);       // NOR

// Conditional (ternary operator) — very useful
assign out = sel ? data1 : data0;    // 2-to-1 MUX

// Arithmetic
assign sum  = a + b;
assign diff = a - b;
assign prod = a * b;          // watch bit widths!

// Concatenation { }
assign bus = {msb, data[6:0]};       // join signals
assign {carry, sum} = a + b + cin;   // split result

// Replication {{}}
assign zeros = {8{1'b0}};            // 8 bits of 0
assign sign_extended = {{24{a[7]}}, a};  // sign extend 8-bit to 32-bit
```

---

## always — Behavioral Blocks

An `always` block describes behavior that executes when a **sensitivity list** event occurs.

### Combinational always block

```verilog
// always @(*) — re-evaluates whenever ANY input changes
always @(*) begin
    if (sel)
        y = a;
    else
        y = b;
end

// Equivalent to:
assign y = sel ? a : b;
```

Use `always @(*)` for combinational logic when you need if/case instead of assign.

### Sequential always block

```verilog
// always @(posedge clk) — executes only on rising clock edge
always @(posedge clk) begin
    if (rst) begin
        counter <= 8'b0;
    end else begin
        counter <= counter + 1;
    end
end

// With asynchronous reset:
always @(posedge clk or posedge rst) begin
    if (rst)
        q <= 1'b0;
    else
        q <= d;
end
```

---

## Blocking vs. Non-Blocking Assignments

This is one of the most important (and most confused) topics in Verilog.

### Blocking Assignment: `=`

Executes immediately. The next line sees the updated value. Use in combinational `always @(*)` blocks.

```verilog
always @(*) begin
    temp = a + b;      // temp gets a+b right now
    y    = temp * 2;   // y sees the updated temp
end
```

### Non-Blocking Assignment: `<=`

Schedules the update. All right-hand sides are evaluated first, then all updates happen simultaneously at the end of the time step. Use in sequential `always @(posedge clk)` blocks.

```verilog
always @(posedge clk) begin
    a <= b;    // a will get b's CURRENT value
    b <= a;    // b will get a's CURRENT value
    // After this block: a and b have swapped
    // (like a simultaneous hardware update)
end
```

**The Golden Rules:**

| Rule | Why |
|------|-----|
| Use `<=` in `always @(posedge clk)` | Models flip-flop behavior correctly |
| Use `=` in `always @(*)` | Models combinational logic correctly |
| Never mix in the same block | Causes simulation/synthesis mismatches |

**What happens if you use `=` in a sequential block?**

```verilog
// WRONG — do not do this
always @(posedge clk) begin
    a = b;    // Updates a immediately
    b = a;    // Sees UPDATED a — not a swap, just a = b = b
end
```

The simulation will work differently from the synthesized hardware. This is a common bug.

---

## if / else

```verilog
always @(*) begin
    if (a == 1'b1)
        y = 1'b1;
    else if (b == 1'b1)
        y = 1'b0;
    else
        y = 1'bx;   // testbench only
end
```

**Important:** In a combinational `always` block, every output must be assigned in every branch. Otherwise, the synthesizer infers a **latch** (memory element) — usually unintended.

```verilog
// BAD — infers a latch for y when sel=0
always @(*) begin
    if (sel)
        y = a;
    // What is y when sel=0? Latch inferred.
end

// GOOD — assign default first
always @(*) begin
    y = 1'b0;   // default
    if (sel)
        y = a;  // override when sel=1
end
```

---

## case Statement

```verilog
// 4-to-1 MUX using case
always @(*) begin
    case (sel)
        2'b00: y = d0;
        2'b01: y = d1;
        2'b10: y = d2;
        2'b11: y = d3;
        default: y = 1'bx;  // always include default
    endcase
end
```

### casez and casex

```verilog
// casez: z (and ?) treated as don't-care
always @(*) begin
    casez (opcode)
        4'b1???: result = op_A;    // MSB=1, others don't care
        4'b01??: result = op_B;
        4'b001?: result = op_C;
        default: result = op_D;
    endcase
end
```

---

## Parameters — Configurable Constants

Parameters make modules reusable.

```verilog
module counter #(
    parameter WIDTH = 8,
    parameter MAX   = 255
) (
    input  wire         clk,
    input  wire         rst,
    output reg [WIDTH-1:0] count
);

always @(posedge clk) begin
    if (rst || count == MAX)
        count <= 0;
    else
        count <= count + 1;
end

endmodule
```

Instantiate with different parameters:
```verilog
// 4-bit counter
counter #(.WIDTH(4), .MAX(9)) ones_digit (
    .clk(clk), .rst(rst), .count(count_ones)
);

// 16-bit counter
counter #(.WIDTH(16), .MAX(999)) big_counter (
    .clk(clk), .rst(rst), .count(count_big)
);
```

---

## Module Instantiation

Connecting modules together:

```verilog
// Named port connection (preferred — clear and order-independent)
full_adder FA0 (
    .a   (A[0]),
    .b   (B[0]),
    .cin (1'b0),
    .sum (S[0]),
    .cout(carry[0])
);

// Another instance
full_adder FA1 (
    .a   (A[1]),
    .b   (B[1]),
    .cin (carry[0]),
    .sum (S[1]),
    .cout(carry[1])
);
```

---

## generate — Repeating Hardware

For repetitive structures, `generate` unrolls a loop at elaboration time (before simulation/synthesis) to create parallel hardware.

```verilog
// Generate an N-bit ripple carry adder
module ripple_adder #(parameter N = 8) (
    input  wire [N-1:0] a, b,
    input  wire         cin,
    output wire [N-1:0] sum,
    output wire         cout
);
    wire [N:0] carry;
    assign carry[0] = cin;
    assign cout = carry[N];

    genvar i;
    generate
        for (i = 0; i < N; i = i + 1) begin : adder_stage
            full_adder FA (
                .a   (a[i]),
                .b   (b[i]),
                .cin (carry[i]),
                .sum (sum[i]),
                .cout(carry[i+1])
            );
        end
    endgenerate
endmodule
```

---

## Operator Reference

```
Bitwise:     &  |  ^  ~  ~&  ~|  ~^
Logical:     &&  ||  !
Reduction:   &a  |a  ^a  (operates on all bits of a)
Shift:       <<  >>  <<<  >>>  (<<< and >>> are arithmetic)
Comparison:  ==  !=  <  >  <=  >=
Equality:    ===  !==  (include x and z in comparison)
Arithmetic:  +  -  *  /  %
Concatenate: {a, b, c}
Replicate:   {N{a}}
Conditional: condition ? true_val : false_val
```

---

## Exercises

Write all of these in Verilog. Save each as a `.v` file.

1. **AND gate**: `module and_gate(input a, b, output y);`
2. **4-bit adder**: Instantiate four full_adder modules in a ripple chain
3. **8-bit register with enable**: Capture D when EN=1 on rising clock edge
4. **4-to-1 MUX**: Using a case statement
5. **Parameterized N-bit shift register**: Shifts right on each clock cycle
6. **Gray code counter**: A 4-bit counter that outputs in Gray code (hint: `gray = binary ^ (binary >> 1)`)

> Please complete [hdlbits](https://hdlbits.01xz.net/wiki/Problem_sets) until the heading more circuits. Without practice, you can not consider yourself knowing verilog.
     


---

**Next:** [Part 3 — Levels of Abstraction](../part3-design-levels/README.md)
