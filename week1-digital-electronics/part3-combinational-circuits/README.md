# Part 3 — Combinational Circuits

> A combinational circuit has no memory. The output depends entirely on the current inputs — nothing else. These are the workhorses of arithmetic and data routing.

---

## What Is a Combinational Circuit?

A **combinational circuit** is one where the output is a **pure function of the current inputs**. There is no internal state, no memory, no feedback. Given the same inputs, you always get the same output.

```
           ┌─────────────────┐
  Inputs──▶│  Combinational  │──▶ Outputs
           │     Logic       │
           └─────────────────┘
```

Examples: adders, subtractors, multiplexers, encoders, decoders, comparators, ALUs.

Contrast this with sequential circuits (Week 2), which have memory and depend on history.

---

## Half Adder

The simplest useful arithmetic circuit. Adds two single bits.

**Inputs:** A, B  
**Outputs:** Sum (S), Carry-out (Cout)

```
Truth Table:
┌───┬───┬───┬──────┐
│ A │ B │ S │ Cout │
├───┼───┼───┼──────┤
│ 0 │ 0 │ 0 │  0   │
│ 0 │ 1 │ 1 │  0   │
│ 1 │ 0 │ 1 │  0   │
│ 1 │ 1 │ 0 │  1   │  ← 1+1=2, which is 10 in binary
└───┴───┴───┴──────┘

Boolean expressions:
  S    = A ⊕ B   (XOR)
  Cout = A · B   (AND)

Circuit:
  A ──┬──[XOR]── S
      │
  B ──┴──[AND]── Cout
```

The half adder is called "half" because it cannot accept a carry-in from a previous stage.

---

## Full Adder

Adds three single bits: A, B, and a Carry-in (Cin). This is what you chain together to add multi-bit numbers.

**Inputs:** A, B, Cin  
**Outputs:** Sum (S), Carry-out (Cout)

```
Truth Table:
┌───┬───┬─────┬───┬──────┐
│ A │ B │ Cin │ S │ Cout │
├───┼───┼─────┼───┼──────┤
│ 0 │ 0 │  0  │ 0 │  0   │
│ 0 │ 0 │  1  │ 1 │  0   │
│ 0 │ 1 │  0  │ 1 │  0   │
│ 0 │ 1 │  1  │ 0 │  1   │
│ 1 │ 0 │  0  │ 1 │  0   │
│ 1 │ 0 │  1  │ 0 │  1   │
│ 1 │ 1 │  0  │ 0 │  1   │
│ 1 │ 1 │  1  │ 1 │  1   │
└───┴───┴─────┴───┴──────┘

Boolean expressions:
  S    = A ⊕ B ⊕ Cin
  Cout = (A · B) + (Cin · (A ⊕ B))

Built from two half adders:
  HA1: A + B  → S1, C1
  HA2: S1 + Cin → S, C2
  Cout = C1 + C2
```

---

## Ripple Carry Adder

To add two N-bit numbers, chain N full adders. The carry-out of each stage feeds into the carry-in of the next.

**Example: 4-bit adder adding A[3:0] + B[3:0]**

```
        A3  B3      A2  B2      A1  B1      A0  B0
         │   │       │   │       │   │       │   │
         ▼   ▼       ▼   ▼       ▼   ▼       ▼   ▼
Cout ◀─[FA3]──C3─▶─[FA2]──C2─▶─[FA1]──C1─▶─[FA0]◀── Cin=0
         │           │           │           │
         ▼           ▼           ▼           ▼
        S3           S2          S1          S0
```

This is called a **ripple carry adder** because the carry "ripples" from LSB to MSB. It is simple but slow for large N, because FA3 cannot compute until FA2 is done, which waits for FA1, etc. 

For a 32-bit ripple carry adder, the critical path passes through 32 full adders. This is why faster designs (Carry-Lookahead Adders, Carry-Select Adders) exist.

---

## Multiplexer (MUX)

A multiplexer is a **data selector**. It selects one of several inputs and routes it to the output, based on a **select** signal.

### 2-to-1 MUX

```
Inputs: I0, I1
Select: S
Output: Y

When S=0: Y = I0
When S=1: Y = I1

Truth Table:
┌───┬────┬────┬───┐
│ S │ I0 │ I1 │ Y │
├───┼────┼────┼───┤
│ 0 │  0 │  X │ 0 │
│ 0 │  1 │  X │ 1 │
│ 1 │  X │  0 │ 0 │
│ 1 │  X │  1 │ 1 │
└───┴────┴────┴───┘

Boolean: Y = S'·I0 + S·I1

Circuit:
  I0 ──[AND]──┐
     └── S' ──┘  └──[OR]── Y
  I1 ──[AND]──┘
     └── S ───┘
```

### 4-to-1 MUX

Requires 2 select lines (S1, S0) to choose among 4 inputs.

```
S1  S0  │  Y
─────────────
 0   0  │  I0
 0   1  │  I1
 1   0  │  I2
 1   1  │  I3

Boolean: Y = S1'·S0'·I0 + S1'·S0·I1 + S1·S0'·I2 + S1·S0·I3
```

**MUX as a universal logic element:** An N-input MUX can implement any function of N variables. Set the data inputs (I0, I1, ...) to the desired output values for each input combination. This is related to how FPGAs implement logic internally using **Look-Up Tables (LUTs)** — more on this in Week 4.

---

## Demultiplexer (DEMUX)

The opposite of a MUX. Takes one input and routes it to one of several outputs based on select lines.

```
1-to-4 DEMUX:

Input: D
Select: S1, S0
Outputs: Y0, Y1, Y2, Y3

S1  S0  │  Active output
─────────────────────────
 0   0  │  Y0 = D
 0   1  │  Y1 = D
 1   0  │  Y2 = D
 1   1  │  Y3 = D
(all other outputs = 0)
```

DEMUXs are used in memory address decoding — the address lines select which memory chip receives the data.

---

## Encoder

An encoder converts one-hot (or other) coded input into a binary code.

### 4-to-2 Priority Encoder

Takes 4 inputs (exactly one HIGH at a time) and outputs the 2-bit binary code of which input is HIGH.

```
┌────┬────┬────┬────┬────┬────┐
│ I3 │ I2 │ I1 │ I0 │ Y1 │ Y0 │
├────┼────┼────┼────┼────┼────┤
│  0 │  0 │  0 │  1 │  0 │  0 │  ← input 0 active
│  0 │  0 │  1 │  0 │  0 │  1 │  ← input 1 active
│  0 │  1 │  0 │  0 │  1 │  0 │  ← input 2 active
│  1 │  0 │  0 │  0 │  1 │  1 │  ← input 3 active
└────┴────┴────┴────┴────┴────┘

Y1 = I2 + I3
Y0 = I1 + I3
```

**Real-world use:** Keyboard encoders convert which key is pressed into a scan code. Interrupt priority encoders determine which hardware interrupt should be handled.

---

## Decoder

A decoder is the inverse of an encoder. Takes a binary code as input and activates one of several outputs.

### 2-to-4 Decoder

```
Inputs: A, B (2 bits → 4 possible values)
Outputs: Y0, Y1, Y2, Y3 (exactly one HIGH at a time)

┌───┬───┬────┬────┬────┬────┐
│ A │ B │ Y0 │ Y1 │ Y2 │ Y3 │
├───┼───┼────┼────┼────┼────┤
│ 0 │ 0 │  1 │  0 │  0 │  0 │
│ 0 │ 1 │  0 │  1 │  0 │  0 │
│ 1 │ 0 │  0 │  0 │  1 │  0 │
│ 1 │ 1 │  0 │  0 │  0 │  1 │
└───┴───┴────┴────┴────┴────┘

Y0 = A'·B'    Y1 = A'·B
Y2 = A·B'     Y3 = A·B
```

**Real-world use:** Memory address decoding — a 3-to-8 decoder takes a 3-bit address and selects one of 8 memory chips. Register file decoding in CPUs.

---

## Comparator

Compares two binary numbers A and B and outputs which is greater, or if equal.

### 1-bit Comparator

```
Outputs: A>B, A=B, A<B

A=B: Y_eq = A XNOR B = A'B' + AB
A>B: Y_gt = A · B'
A<B: Y_lt = A'· B
```

### 4-bit Comparator

Start from the MSB. If MSBs differ, that determines the comparison. If equal, move to next bit, and so on. This can be implemented as a cascaded chain of 1-bit comparators.

---

## Arithmetic Logic Unit (ALU)

An ALU combines multiple arithmetic and logical operations into one circuit, selectable by a control signal.

```
                  Operation select
                       │
  A[N:0] ──────────────┤
                    [ALU]     ──── Result[N:0]
  B[N:0] ──────────────┤      ──── Zero flag
                              ──── Carry flag
                              ──── Overflow flag

Common operations (selected by 3-4 bit code):
  000: A + B     (add)
  001: A - B     (subtract)
  010: A AND B
  011: A OR B
  100: A XOR B
  101: NOT A
  110: A << 1    (shift left)
  111: A >> 1    (shift right)
```

The ALU is the heart of every CPU. Your FPGA will contain one if you implement a CPU, or you can instantiate the built-in DSP slices (dedicated multiplier-adder hardware).

---

## Video Resources

- [Half Adder and Full Adder](https://www.youtube.com/watch?v=5XbRIVWFRIw)
- [Multiplexers](https://www.youtube.com/watch?v=aQlF-9i3fAA)
- [Encoders and Decoders](https://www.youtube.com/watch?v=feBvhLFQEDk&pp=ygUTZW5jb2RlciBhbmQgZGVjb2RlctIHCQkjCwGHKiGM7w%3D%3D)
- [How does an ALU work?](https://www.youtube.com/watch?v=1I5ZMmrOfnA)

---

## Week 1 Summary

You now understand:
- How binary represents all digital information
- How transistors implement logic gates
- How Boolean algebra lets you simplify circuits
- How combinational circuits like adders, MUXes, and decoders work

These are the atomic building blocks. Everything from this point forward is composed from these pieces.

---

## Week 1 Exercises

Build these on paper (truth tables + equations) before writing any code:

1. **Design a 1-bit subtractor** (inputs: A, B, Borrow-in; outputs: Difference, Borrow-out)
2. **Design an 8-to-3 encoder** (8 one-hot inputs → 3-bit binary output)
3. **Design a 1-bit comparator** with outputs EQ, GT, LT
4. **Design a 4-to-1 MUX** using only 2-to-1 MUXes
5. **Implement the majority function** Y = AB + BC + AC using only NAND gates

---

**Next:** [Part 4 — CMOS Timings hazards](../part4-cmos-timing-hazards/README.md)