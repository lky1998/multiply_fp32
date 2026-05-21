# fmultiplier — FP32 Multiplier (7-Cycle Sequential, Handshake-Based)

## Overview
`fmultiplier` is a single-issue, multi-cycle single-precision floating-point multiplier. It accepts one operation at a time using a `valid` / `out_valid` handshake and produces a 32-bit IEEE-754 binary32 result.

This design must:
- produce bit-accurate results for normal FP32 numbers,
- use round-to-nearest-even (RNE),
- have fixed latency of exactly 7 clock cycles from input acceptance to output valid,
- compute `z = a * b`, where `a`, `b`, and `z` are 32-bit IEEE-754 single-precision values.

---

## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk` | in | 1 | Clock |
| `rst` | in | 1 | Asynchronous reset, active high |
| `valid` | in | 1 | 1-cycle start pulse; accepted only when not busy |
| `a` | in | 32 | Operand A (FP32 bits) |
| `b` | in | 32 | Operand B (FP32 bits) |
| `z` | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | 1-cycle pulse when `z` is valid |

---

## IEEE-754 FP32 Format
Bit layout:
- `[31]` = sign bit
- `[30:23]` = 8-bit biased exponent
- `[22:0]` = 23-bit fraction

For normal numbers:
- mantissa = `{1'b1, frac[22:0]}` (24 bits total)
- unbiased exponent = `exp - 127`

---

## Essential Edge Cases

### Zero / Subnormal Input Detection
If either input has `exp == 0`:
- result must be zero
- result sign = `a_sign ^ b_sign`

### Overflow Detection
If the final biased exponent is `>= 255`:
- result = infinity
- result sign = `a_sign ^ b_sign`

### Underflow Detection
If the final biased exponent is `<= 0`:
- result = zero
- result sign = `a_sign ^ b_sign`

---

## Handshake Contract

### Start condition
- When `busy == 0`, a high `valid` sampled on a rising clock edge starts a new operation.
- On that edge:
  - `a` and `b` are latched internally,
  - `busy` is asserted,
  - the 7-cycle internal operation begins.

### Busy behavior
- While `busy == 1`, the unit is processing the current operation.
- Any `valid` asserted while `busy == 1` must be ignored.

### Completion condition
- `out_valid` must assert exactly 7 rising clock edges after the start edge.
- `z` must be valid on the same cycle as `out_valid`.
- `busy` must be deasserted after the result is produced.

### Timing definition
- The clock edge that samples `valid == 1` while idle is cycle 0.
- `out_valid` must pulse on cycle 7.

---

## Latency and Throughput

### Latency
- Fixed latency: 7 cycles
- Start edge to `out_valid`: exactly 7 rising edges

### Throughput
- Single-issue design
- Maximum throughput is 1 result every 7 cycles

---

## Algorithm Overview

### Core Multiplication Steps
1. Extract sign, exponent, and fraction from both operands
2. Check edge cases first
3. Build mantissas as `{1'b1, frac[22:0]}` for normal numbers
4. Multiply mantissas: 24 × 24 = 48-bit product
5. Normalize the product
6. Round using round-to-nearest-even
7. Pack sign, exponent, and fraction into IEEE-754 format

---

## Internal Representation

For each operand:
- `a_s`, `b_s` = sign bits
- `a_e`, `b_e` = biased exponents
- `a_f`, `b_f` = fractions
- `a_m`, `b_m` = 24-bit mantissas with hidden 1 for normal numbers
- `product` = 48-bit mantissa product
- `guard_bit`, `round_bit`, `sticky_bit` = rounding support bits

---

## Precise Implementation Requirements

### Mantissa Multiplication
Must use 48-bit result:
