# Part 2 — Finite State Machines

> Every digital system with behavior — a vending machine, a traffic light, a UART receiver, a CPU — is a finite state machine. This is one of the most powerful ideas in engineering.

---

## What Is a Finite State Machine?

A **Finite State Machine (FSM)** is a computational model with:

- A **finite** set of states
- A **current state** (stored in flip-flops)
- **Transitions** between states, triggered by inputs
- **Outputs** produced by the machine

At every clock edge, the FSM evaluates its current state and current inputs, transitions to a new state, and produces outputs.

```
                 ┌───────────────┐
                 │  Next-State   │
Inputs ────────▶│    Logic      │──── Next State
                 │(combinational)│          │
                 └───────────────┘          │
                        ▲                   ▼
                        │          ┌──────────┐
Current State ──────────┤          │ State    │
                        │          │ Register │
                        └──────────│ (FFs)    │
                                   └──────────┘
                                        │
                                        ▼ (current state)
                                ┌───────────────┐
                                │     Output    │
                                │     Logic     │────── Outputs
                                │(combinational)│
                                └───────────────┘
```

This is the **general FSM architecture**. The state register holds the current state. Combinational logic computes the next state and the outputs.

---

## Mealy vs. Moore Machines

There are two FSM types, differing in how outputs are produced:

### Moore Machine

Outputs depend **only on the current state**.

```
Output = f(current_state)
```

Advantages: outputs are stable (change only at clock edges), easier to analyze, less prone to glitches.

### Mealy Machine

Outputs depend on **current state AND current inputs**.

```
Output = f(current_state, inputs)
```

Advantages: often requires fewer states, responds faster (output can change within a clock cycle when inputs change).

**Which to use?** In practice, Moore is safer and more common in FPGA design. Mealy can be more efficient when you need output to react immediately to inputs.

---

## State Diagrams

A **state diagram** (bubble diagram) visualizes an FSM:

- **Circles** = states
- **Arrows** = transitions
- Labels on arrows = condition (input that causes transition)
- Labels inside or beside states = outputs

**Example: Traffic Light Controller**

States: RED, GREEN, YELLOW  
The light cycles through states based on a timer.

```
        [timer expired]           [timer expired]
  ┌──────────────────┐      ┌──────────────────┐
  │                  ▼      │                  ▼
[RED]───────────────▶[GREEN]─────────────────▶[YELLOW]
  ▲                                            │
  │               [timer expired]              │
  └────────────────────────────────────────────┘

Outputs (Moore):
  RED    state → RED light on
  GREEN  state → GREEN light on
  YELLOW state → YELLOW light on
```

---

## Example 1: Sequence Detector

**Problem:** Detect the sequence "1011" in a serial bit stream. Output HIGH when the last 4 bits received are 1, 0, 1, 1.

**States:**
- S0: Initial (or last bit didn't start sequence)
- S1: Received "1"
- S2: Received "10"
- S3: Received "101"
- S4: Received "1011" → OUTPUT HIGH, return to check

**State Diagram:**

```
[S0]
 ▲
 │ 0 (self-loop)                
 │                  
[S0] ──1──▶ [S1] ──0──▶ [S2] ──1──▶ [S3] ──1──▶ [S4/OUT]
 ▲            │                        │              │
 │            │ 1 (self-loop)          │ 0            │ 0
 │            ▼                        ▼              │
 │           [S1]                    [S0]             │
 │                                                    │ 1
 └────────────────────────────────────────────────────┘
```

**State Transition Table (Mealy, output=1 only in S4 transition):**

| Current State | Input | Next State | Output |
|---------------|-------|------------|--------|
| S0 | 0 | S0 | 0 |
| S0 | 1 | S1 | 0 |
| S1 | 0 | S2 | 0 |
| S1 | 1 | S1 | 0 |
| S2 | 0 | S0 | 0 |
| S2 | 1 | S3 | 0 |
| S3 | 0 | S2 | 0 |
| S3 | 1 | S0 | 1 ← detected! |

---

## Example 2: Vending Machine Controller

**Scenario:** A vending machine accepts 5¢ and 10¢ coins. Item costs 15¢. When enough money is deposited, dispense item.

**States:** S0 (0¢), S5 (5¢), S10 (10¢), DISPENSE

```
 5¢               5¢               5¢
     [S0] ────────▶ [S5] ────────▶ [S10] ────────▶ [DISP] ──▶ (back to S0)  
      │                             ▲                ↑
      │ 10¢             10¢         │                │
      └─────────────────────────────┘────────────────┘
                                    ↑
                               [S10] + 10¢ also goes here             
```

**Output logic (Moore):**
- States S0, S5, S10: no output
- State DISPENSE: activate dispense motor, return to S0

---

## FSM Design Process

Follow this process every time you design an FSM:

1. **Understand the problem** — what are the inputs, outputs, and behaviors?
2. **Identify all states** — what distinct "situations" can the system be in?
3. **Draw the state diagram** — all states, all transitions, outputs
4. **Write the state transition table**
5. **Choose state encoding** (binary, one-hot, Gray code)
6. **Derive next-state logic** (combinational equations or just use case statement in Verilog)
7. **Implement in Verilog** (Week 3 will cover this)
8. **Simulate and verify**

---

## State Encoding

How you assign binary codes to states matters:

### Binary Encoding

Minimum flip-flops. N states need ⌈log₂N⌉ flip-flops.

```
4 states → 2 flip-flops:
  S0 = 00
  S1 = 01
  S2 = 10
  S3 = 11
```

More complex next-state logic. Good for minimizing area.

### One-Hot Encoding

One flip-flop per state. Exactly one flip-flop is HIGH at a time.

```
4 states → 4 flip-flops:
  S0 = 0001
  S1 = 0010
  S2 = 0100
  S3 = 1000
```

Simpler next-state logic (the transitions are just direct connections). Faster. Uses more flip-flops. **Preferred for FPGAs** because FPGAs have abundant flip-flops and the simplified logic reduces LUT usage.

### Gray Code Encoding

Adjacent states differ by exactly one bit. Reduces glitches in outputs.

```
4 states:
  S0 = 00
  S1 = 01
  S2 = 11
  S3 = 10
```

Used in counters driving external interfaces to prevent momentary wrong values during transitions.

---

## The Three-Process FSM Pattern in Verilog

When you implement FSMs in Verilog (Week 3), the standard clean pattern uses three separate `always` blocks:

```verilog
// Process 1: State register (sequential)
always @(posedge clk or posedge rst) begin
    if (rst)
        state <= S0;
    else
        state <= next_state;
end

// Process 2: Next-state logic (combinational)
always @(*) begin
    case (state)
        S0: if (input_x) next_state = S1;
            else          next_state = S0;
        S1: ...
        // all states
    endcase
end

// Process 3: Output logic (combinational, Moore)
always @(*) begin
    case (state)
        S0: output_y = 0;
        S1: output_y = 1;
        // ...
    endcase
end
```

This separation is not just style — it prevents common mistakes like inadvertently creating latches, and makes timing analysis and debugging straightforward.

---

## Common FSM Patterns in Hardware

| Pattern | Example Use |
|---------|-------------|
| Sequence detector | Protocol framing (start/stop bits) |
| Counter with conditions | Wait states in memory controllers |
| Handshake protocol | AXI valid/ready signals |
| Traffic controller | Any multi-phase operation |
| UART receiver | Detecting start bit, sampling data bits, stop bit |
| SPI controller | Generating CS, CLK, MOSI in sequence |
| Debouncer | Ignoring bounce in mechanical switches |

---

## FSM Pitfalls

### 1. Missing Transitions

If your state diagram doesn't handle every possible input in every state, hardware will infer latches or go to undefined states. Always have a `default` transition.

### 2. Output Glitches (Mealy machines)

Mealy outputs can glitch when inputs change asynchronously relative to the clock. Register the outputs if glitches cause problems.

### 3. Unreachable States

With binary encoding, there may be more bit patterns than states (e.g., 3 states with 2 bits → 4 possible patterns). Add transitions from the unused state to a safe default.

### 4. Reset State

Always define which state the FSM enters on reset. Without this, the initial state after power-on is undefined.

---


## Week 2 Exercises

1. Design an FSM for a **combination lock**: The lock opens only if the sequence 0-1-1-0 is entered. Any wrong input resets. Draw the state diagram and write the transition table.

2. Design a **sequence detector** for "101" (overlapping — "10101" should detect twice).

3. A **simple CPU control unit** goes through: FETCH → DECODE → EXECUTE → WRITEBACK → FETCH. Add a STALL state entered from EXECUTE when a memory signal is HIGH, returning to EXECUTE on the next cycle. Draw the FSM.

4. For the traffic light example, add a PEDESTRIAN state entered when a button is pressed during GREEN. After 5 seconds (assume a counter provides a tick), return to the normal cycle.

5. Implement the vending machine FSM as a state transition table. Use binary encoding and derive the Boolean equations for the next-state logic.

---

**Week 3:** [Verilog HDL — Writing Hardware as Code](../../week3-verilog/README.md)
