
# Part 4 — Simulation with iverilog & GTKWave

## Why Simulate?

Simulation lets you test your design **before touching an FPGA**. This is critical because:

- Debugging on hardware is much harder than in simulation
- Simulation runs fast — you can test thousands of cycles in seconds
- Simulation can show internal signals that are invisible on hardware
- Simulation can inject specific test cases including edge cases

**The rule:** Simulate until your design is correct. Then synthesize.

---

## Installing iverilog and GTKWave

### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install iverilog gtkwave
```

### macOS

```bash
brew install icarus-verilog gtkwave
```

### Windows

Download the installer from:
- iverilog: http://bleyer.org/icarus/ (Windows binary)
- GTKWave: bundled with the iverilog installer, or https://gtkwave.sourceforge.net/

Or use WSL (Windows Subsystem for Linux) and follow the Linux instructions.

---

## Your First Simulation

### Step 1: Write the Design

```verilog
// half_adder.v
module half_adder (
    input  wire a, b,
    output wire sum, cout
);
    assign sum  = a ^ b;
    assign cout = a & b;
endmodule
```

### Step 2: Write the Testbench

A **testbench** is a Verilog module that has no ports. It instantiates your design, drives inputs, and checks outputs.

```verilog
// half_adder_tb.v
`timescale 1ns/1ps   // time unit / time precision

module half_adder_tb;

    // Declare signals to connect to DUT (Design Under Test)
    reg  a, b;          // reg because we drive them in initial block
    wire sum, cout;     // wire because the DUT drives them

    // Instantiate the Design Under Test
    half_adder DUT (
        .a   (a),
        .b   (b),
        .sum (sum),
        .cout(cout)
    );

    // Dump waveform for GTKWave
    initial begin
        $dumpfile("half_adder_tb.vcd");
        $dumpvars(0, half_adder_tb);
    end

    // Apply stimulus
    initial begin
        // Apply all 4 input combinations
        a = 0; b = 0;
        #10;              // wait 10 ns

        a = 0; b = 1;
        #10;

        a = 1; b = 0;
        #10;

        a = 1; b = 1;
        #10;

        $finish;          // end simulation
    end

    // Monitor outputs
    initial begin
        $monitor("Time=%0t a=%b b=%b | sum=%b cout=%b",
                 $time, a, b, sum, cout);
    end

endmodule
```

### Step 3: Compile and Simulate

```bash
# Compile
iverilog -o half_adder_sim half_adder.v half_adder_tb.v

# Run simulation
./half_adder_sim

# Expected output:
# Time=0 a=0 b=0 | sum=0 cout=0
# Time=10 a=0 b=1 | sum=1 cout=0
# Time=20 a=1 b=0 | sum=1 cout=0
# Time=30 a=1 b=1 | sum=0 cout=1
```

### Step 4: View Waveforms in GTKWave

```bash
gtkwave half_adder_tb.vcd
```

In GTKWave:
1. Expand the module tree on the left
2. Select signals you want to view
3. Click "Append" or drag them to the wave window
4. Click the zoom-fit button (or press Ctrl+F)

---

## Testbench Patterns

### Clock Generation

```verilog
// 100 MHz clock (10 ns period)
parameter CLK_PERIOD = 10;
reg clk;

initial clk = 0;
always #(CLK_PERIOD/2) clk = ~clk;    // toggle every 5 ns
```

### Reset Sequence

```verilog
initial begin
    rst = 1;
    @(posedge clk);    // wait for rising edge
    @(posedge clk);    // hold reset for 2 cycles
    rst = 0;
    // design now running
end
```

### Checking Outputs (Assertions)

```verilog
task check_output;
    input [7:0] expected;
    begin
        if (out !== expected) begin
            $display("FAIL: Expected %h, got %h at time %0t",
                     expected, out, $time);
            $finish;
        end else begin
            $display("PASS: out = %h", out);
        end
    end
endtask

initial begin
    // ... apply inputs ...
    #10;
    check_output(8'hAB);
end
```

### `$monitor` vs `$display` vs `$strobe`

| Task | When it executes |
|------|-----------------|
| `$display` | Immediately when the line is reached |
| `$monitor` | Whenever any listed signal changes (continuous) |
| `$strobe` | At the end of the current time step |
| `$write` | Same as $display but no newline |

---

## Complete Example: FSM Testbench

```verilog
// traffic_light_tb.v
`timescale 1ns/1ps

module traffic_light_tb;

    reg  clk, rst;
    wire [1:0] state;
    wire red, green, yellow;

    // DUT
    traffic_light DUT (
        .clk(clk), .rst(rst),
        .state(state),
        .red(red), .green(green), .yellow(yellow)
    );

    // Clock: 10 ns period
    initial clk = 0;
    always #5 clk = ~clk;

    // Waveform dump
    initial begin
        $dumpfile("traffic_light_tb.vcd");
        $dumpvars(0, traffic_light_tb);
    end

    // Test sequence
    initial begin
        rst = 1;
        repeat(2) @(posedge clk);
        rst = 0;

        // Run for 100 clock cycles
        repeat(100) @(posedge clk);

        $display("Simulation complete.");
        $finish;
    end

    // State monitor
    always @(posedge clk) begin
        $display("Cycle %0t: state=%0d R=%b G=%b Y=%b",
                 $time, state, red, green, yellow);
    end

endmodule
```

---

## Simulation vs. Synthesis Mismatches — Common Pitfalls

These will cost you hours if you are not aware of them:

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Using `=` in sequential block | Sim correct, hardware wrong | Use `<=` |
| Missing `default` in case | Latch inferred | Always add `default` |
| Incomplete sensitivity list | `always @(a)` missing `b` | Use `always @(*)` |
| `initial` block in design | Ignored by synthesizer | Use reset logic |
| Division/modulo | Works in sim, may not synthesize | Avoid or use power-of-2 dividers |

---

## Exercises

1. Write and simulate a **full adder** — verify all 8 input combinations
2. Write a testbench for a **4-bit counter** — verify it counts 0→15 and wraps to 0
3. Write a testbench for a **2-to-1 MUX** — verify both select cases
4. Write a testbench for an **8-bit shift register** — load 8'hA5 and shift 8 times, check output each cycle
5. Write a testbench for a **sequence detector** (from Week 2) — apply the sequence "1011" embedded in a longer stream and verify detection

> Please complete [hdlbits](https://hdlbits.01xz.net/wiki/Problem_sets) full problem set. Without practice, you can not consider yourself knowing verilog.
     

---

**Next:** [Part 5 — Verilog Traps, Inference Rules & Everything the Synthesizer Does](../part5-verilog-traps-deep/README.md )

