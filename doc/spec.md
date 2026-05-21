# fmultiplier — FP32 Multiplier (7-Cycle Sequential, Handshake-Based)

## Overview
`fmultiplier` is a single-issue, multi-cycle single-precision floating-point multiplier. It accepts one operation at a time using a `valid` / `out_valid` handshake and produces a 32-bit IEEE-754 binary32 result.

This design is intended to:
- produce bit-accurate results for normal FP32 numbers,
- use round-to-nearest-even (RNE),
- have fixed latency of exactly 7 clock cycles from input acceptance to output valid,
- compute `z = a * b`, where `a`, `b`, and `z` are 32-bit IEEE-754 single-precision values.

---

## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`   | in  | 1  | Clock |
| `rst`   | in  | 1  | Asynchronous reset, active high |
| `valid` | in  | 1  | 1-cycle start pulse; accepted only when not busy |
| `a`     | in  | 32 | Operand A (FP32 bits) |
| `b`     | in  | 32 | Operand B (FP32 bits) |
| `z`     | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | 1-cycle pulse when `z` is valid |

---

## CRITICAL: IEEE-754 FP32 Format
Bit Layout: [31] [30:23] [22:0] = [S] [EXP] [FRAC]
- Bit 31: Sign bit (0=positive, 1=negative)
- Bits 30:23: 8-bit biased exponent (bias = 127)
- Bits 22:0: 23-bit fraction (mantissa without hidden 1)

For normal numbers:
- Actual mantissa = {1'b1, frac[22:0]} (24 bits total)
- Actual exponent = exp - 127 (unbiased)

---

## CRITICAL: Essential Edge Cases
While normal numbers are the primary focus, you MUST handle these cases correctly:

### Zero Input Detection
If either input has exp field = 0 (zero or subnormal):
- Result = {result_sign, 8'h00, 23'h000000} (zero)
- Result sign = a_sign ^ b_sign

### Overflow Detection
If the final biased exponent ≥ 255:
- Result = {result_sign, 8'hFF, 23'h000000} (infinity)
- Result sign = a_sign ^ b_sign

### Underflow Detection  
If the final biased exponent ≤ 0:
- Result = {result_sign, 8'h00, 23'h000000} (zero)
- Result sign = a_sign ^ b_sign

---

## Handshake Contract

### Start condition
- When `busy == 0`, a high `valid` sampled on a rising clock edge starts a new operation.
- On that edge:
  - `a` and `b` are registered internally,
  - `busy` is asserted,
  - the 7-cycle internal operation begins.

### Busy behavior
- While `busy == 1`, the unit is busy processing the current operation.
- Any `valid` asserted while busy is high is ignored.

### Completion condition
- `out_valid` must assert exactly 7 rising clock edges after the start edge.
- `z` must be valid on the same cycle as `out_valid`.
- `busy` is deasserted after the result is produced.

### Timing definition
To avoid ambiguity:
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
1. **Extract fields**: sign, exponent, fraction from both operands
2. **Check for edge cases**: zero inputs, potential overflow/underflow
3. **Build mantissas**: {1'b1, frac[22:0]} for normal numbers
4. **Multiply mantissas**: 24×24 = 48-bit product
5. **Normalize**: adjust for leading 1 position
6. **Round**: apply round-to-nearest-even
7. **Pack result**: combine sign, exponent, fraction

---

## Internal Representation

For each operand:
- `sign` = bit 31
- `exp` = bits 30:23
- `frac` = bits 22:0

Internal signals:
- `a_s`, `b_s`, `z_s` : sign bits
- `a_e`, `b_e`, `z_e` : signed unbiased exponents
- `a_m`, `b_m` : 24-bit mantissas with hidden 1 for normal numbers
- `product` : mantissa product (MUST be 48-bit wide for full precision)
- `guard_bit`, `round_bit`, `sticky` : rounding support bits

---

## CRITICAL: Precise Implementation Requirements

### Mantissa Multiplication MUST Use 48-bit Result
```verilog
wire [47:0] product = a_mant * b_mant;  // 24×24 = 48 bits
