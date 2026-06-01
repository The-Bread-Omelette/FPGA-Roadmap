# Part 1 — Number Systems

> Before you can build hardware, you need to understand the language hardware speaks. That language is binary.

---

## Why Does Hardware Use Binary?

A transistor — the fundamental building block of all modern electronics — is a switch. It is either ON or OFF. There is no "half on." This physical reality means every piece of information inside a computer is stored as a sequence of ONs and OFFs. We represent ON as **1** and OFF as **0**.

This is the binary (base-2) number system.

---

## The Decimal System You Already Know

In the decimal (base-10) system, each digit position represents a power of 10:

```
  4  3  2  1  0    ← position
┌──┬──┬──┬──┬──┐
│  │  │  │  │  │
└──┴──┴──┴──┴──┘
 10⁴ 10³ 10² 10¹ 10⁰
```

The number **2,473** means:

```
2 × 10³  +  4 × 10²  +  7 × 10¹  +  3 × 10⁰
= 2000   +   400     +    70     +     3
= 2473
```

---

## The Binary System

Binary works exactly the same way, but each position represents a power of **2** instead of 10:

```
  7    6    5    4    3    2    1    0    ← bit position
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 1  │ 0  │ 1  │ 1  │ 0  │ 1  │
└────┴────┴────┴────┴────┴────┴────┴────┘
 2⁷   2⁶   2⁵   2⁴   2³   2²   2¹   2⁰
 128   64   32   16    8    4    2    1
```

Reading the example: `0110 1101`

```
0×128 + 1×64 + 1×32 + 0×16 + 1×8 + 1×4 + 0×2 + 1×1
=   0 +   64 +   32 +    0 +   8 +   4 +   0 +   1
= 109
```

So binary `01101101` = decimal `109`.

---

## Terminology

| Term | Meaning |
|------|---------|
| **Bit** | A single binary digit (0 or 1) |
| **Byte** | 8 bits |
| **Word** | Typically 16, 32, or 64 bits depending on the architecture |
| **MSB** | Most Significant Bit — the leftmost bit (highest value) |
| **LSB** | Least Significant Bit — the rightmost bit (lowest value) |

---

## Converting Decimal → Binary

**Method: Repeated Division by 2**

Divide the decimal number by 2 repeatedly. The remainders (read bottom to top) give the binary representation.

**Example: Convert 45 to binary**

```
45 ÷ 2 = 22  remainder 1   ← LSB
22 ÷ 2 = 11  remainder 0
11 ÷ 2 =  5  remainder 1
 5 ÷ 2 =  2  remainder 1
 2 ÷ 2 =  1  remainder 0
 1 ÷ 2 =  0  remainder 1   ← MSB

Reading remainders bottom to top: 101101
```

So decimal `45` = binary `101101`

**Verify:** 1×32 + 0×16 + 1×8 + 1×4 + 0×2 + 1×1 = 32+8+4+1 = **45** ✓

---

## Converting Binary → Decimal

**Method: Positional weights**

Simply multiply each bit by its positional value and add.

**Example: Convert `10110011` to decimal**

```
Bit:      1    0    1    1    0    0    1    1
Position: 7    6    5    4    3    2    1    0
Value:   128   0   32   16    0    0    2    1

Sum = 128 + 32 + 16 + 2 + 1 = 179
```

---

## Hexadecimal — A Shorthand for Binary

Binary strings get long fast. An 8-bit number requires writing 8 digits. A 32-bit address requires 32 digits. This is tedious.

**Hexadecimal (base-16)** solves this by grouping 4 bits into a single digit. Since base-16 needs 16 symbols, we use 0–9 and A–F:

| Decimal | Binary | Hex |
|---------|--------|-----|
| 0 | 0000 | 0 |
| 1 | 0001 | 1 |
| 2 | 0010 | 2 |
| 3 | 0011 | 3 |
| 4 | 0100 | 4 |
| 5 | 0101 | 5 |
| 6 | 0110 | 6 |
| 7 | 0111 | 7 |
| 8 | 1000 | 8 |
| 9 | 1001 | 9 |
| 10 | 1010 | A |
| 11 | 1011 | B |
| 12 | 1100 | C |
| 13 | 1101 | D |
| 14 | 1110 | E |
| 15 | 1111 | F |

Hex numbers are written with a `0x` prefix: `0x1F`, `0xABCD`.

**Example: Convert binary `1010 1111 0011 0110` to hex**

Group into 4-bit nibbles from right:
```
1010   1111   0011   0110
  A      F      3      6

Result: 0xAF36
```

This is why memory addresses look like `0x00400000` instead of a 32-digit binary string.

---

## Binary Arithmetic

### Addition

Binary addition follows the same rules as decimal:

```
Rules:
0 + 0 = 0
0 + 1 = 1
1 + 0 = 1
1 + 1 = 0, carry 1
1 + 1 + 1 (carry) = 1, carry 1
```

**Example:**

```
  0 1 1 0 1   (13)
+ 0 1 0 1 1   (11)
-----------
  1 1 0 0 0   (24)
```

Carry propagation:
```
  carries: 1 1 1 1 0
    01101
  + 01011
  -------
    11000
```

This is exactly how digital adder circuits work — we will build one in Part 3.

### Two's Complement (Signed Numbers)

How do we represent negative numbers in binary? The standard method is **two's complement**.

For an N-bit number:
- The MSB (leftmost bit) is the **sign bit**: 0 = positive, 1 = negative
- To negate a number: **flip all bits, then add 1**

**Example: Represent -5 in 8-bit two's complement**

```
Step 1: Write +5 in binary:    0000 0101
Step 2: Flip all bits:         1111 1010
Step 3: Add 1:                 1111 1011

-5 in 8-bit two's complement = 1111 1011 = 0xFB
```

**Verification:** 1111 1011
- As unsigned: 128+64+32+16+8+0+2+1 = 251
- As signed (two's complement): 251 - 256 = **-5** ✓

Why does this work? With two's complement, you can add positive and negative numbers using the same addition hardware. No special subtraction circuit is needed. This elegance is why every CPU and FPGA uses this representation. 
Additional Conversion about float can be learnt in ESC111, also you can refer how division, multiplication, etc work in a computer on the internet.

---

## Digital vs. Analog Signals

The real world is **analog** — voltage levels, temperatures, and sounds are continuous values. Digital circuits work with **discrete** levels.

```
Analog signal:              Digital signal:
     │                           │
  3V │  ~~~~                  3V │▔▔▔▔│    │▔▔▔▔▔▔▔
     │      ~~~~                 │    │    │
  0V │──────────              0V │────┘    └───────
     └──────────                 └──────────────────
           time                        time
```

In a digital circuit, any voltage above a threshold is read as **1** (HIGH), and any voltage below another threshold is read as **0** (LOW). This makes digital circuits immune to small amounts of noise — a signal that is supposed to be 3.3V but is actually 3.1V due to interference still reads as HIGH.

**Logic voltage levels vary by technology:**

| Standard | HIGH | LOW | Used In |
|----------|------|-----|---------|
| TTL | 2.4V – 5V | 0V – 0.8V | Older logic chips |
| CMOS 3.3V | 2.0V – 3.3V | 0V – 0.8V | Modern FPGAs (common) |
| CMOS 1.8V | 1.0V – 1.8V | 0V – 0.5V | Low-power FPGAs |
| LVDS | Differential pair | — | High-speed serial |

---

## Video Resources

For additional visual explanations of number systems:

- [Introduction to Number Systems](https://www.youtube.com/watch?v=FFDMzbrEXaE)
- [Why Binary?](https://www.youtube.com/watch?v=thrx3SBEpL8)

---

## Exercises

Before moving on, practice these until they feel automatic:

1. Convert the following to binary: 37, 100, 255, 1024
2. Convert the following binary numbers to decimal: `110101`, `00001111`, `10000001`
3. Convert `0xDEAD` to binary (group into 4-bit nibbles)
4. What is the two's complement representation of -13 in 8 bits?
5. Add binary `01101` and `01011`. What decimal numbers are these, and does your binary sum match?

---

**Next:** [Part 2 — Logic Gates, Boolean Algebra & Transistors](../part2-logic-gates/README.md)
