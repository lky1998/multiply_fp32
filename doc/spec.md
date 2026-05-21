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

## Target Operand Scope

### Primary target
The primary verification target is normal FP32 operands:
- `exp ∈ [1..254]`
- operands are finite, normal values
- standard IEEE-754 sign handling applies
- rounding mode is round-to-nearest-even

### Out-of-scope for primary grading
Unless explicitly tested, the following are not required for full credit:
- NaN, Infinity, Zero, Subnormals / denormals

If special-case behavior is implemented, it must not break the normal-number path.

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

## Recommended Implementation Order
To keep the design stable and reduce debugging mistakes, implement in this order:

1. Handshake / FSM only
2. Normal-number sign handling
3. Exponent extraction and unbiased conversion
4. 24-bit mantissa formation with hidden 1
5. Mantissa multiplication (48-bit precision)
6. Normalization
7. Round-to-nearest-even
8. Pack result

Do not overcomplicate special-case behavior unless required by tests.

---

## FSM / Pipeline Stages

The design uses a 7-stage internal sequence controlled by `busy` and `counter`.

### Stage 1 — Register inputs
- Sample `a` and `b`
- Capture sign, exponent, and fraction fields
- Initialize internal registers

### Stage 2 — Build mantissas and exponents
- For normal operands, form 24-bit mantissas as `{1'b1, frac}`
- Convert biased exponent to unbiased exponent by subtracting 127

### Stage 3 — Compute result sign and core multiply setup
- Compute result sign: `z_s = a_s ^ b_s`
- Prepare exponent sum and mantissa multiplication

### Stage 4 — Multiply mantissas
- Compute the mantissa product using a 48-bit wide register (24×24=48 bits)
- Compute the raw exponent sum

### Stage 5 — Normalize product and extract rounding bits
- Normalize the product so the leading 1 is in the expected position
- Extract: result mantissa, guard bit, round bit, sticky bit
- If product[47] == 1: shift right by 1, increment exponent
- Extract guard/round/sticky bits based on normalization

### Stage 6 — Round to nearest even
- Apply RNE: if `guard_bit == 1` and (`round_bit == 1` or `sticky == 1` or LSB == 1), increment mantissa
- If rounding causes mantissa overflow, renormalize and increment exponent

### Stage 7 — Pack result
- Convert the exponent back to biased form
- Pack `z = {z_s, biased_exp, fraction}`
- Assert `out_valid` for 1 cycle
- Clear `busy`

---

## CRITICAL: Precise Rounding Implementation

### Mantissa Multiplication MUST Use 48-bit Result
The mantissa multiplication MUST produce a full 48-bit result. Use: `product = a_mant * b_mant` where product is 48-bit wide.

### Rounding Bit Extraction
The rounding decision depends on three bits below the final mantissa:
- Guard bit: First bit below mantissa
- Round bit: Second bit below mantissa  
- Sticky bit: OR of all remaining lower bits

### Round-to-Nearest-Even (RNE) Rule
Round up if: `guard_bit == 1` AND (`round_bit == 1` OR `sticky_bit == 1` OR `lsb == 1`)

This ensures:
- Exact halfway cases round to even (LSB = 0)
- Non-exact cases round to nearest

### Normalization Logic
After multiplication, check if product[47] == 1:
- If yes: result ≥ 2.0, shift right by 1, increment exponent
- If no: result < 2.0, no shift needed

Extract rounding bits accordingly:
- Normalized: mantissa=product[46:24], guard=product[23], round=product[22], sticky=|product[21:0]
- Not normalized: mantissa=product[45:23], guard=product[22], round=product[21], sticky=|product[20:0]

---

## Normal-Number Behavior
For normal inputs:
- Hidden leading 1 is always present in the mantissa
- The result should follow standard FP32 multiplication rules
- RNE must be applied to the final normalized mantissa
- Sign is XOR of input signs

---

## Special-Case Policy
To reduce ambiguity:
- The grading focus is normal-number multiplication
- Special-case support is optional unless required by tests
- Do not let special-case logic disturb the normal path

If special cases are implemented, they should be handled cleanly and separately from the normal path.

---

## Verification Notes
Recommended testbench usage:
- Drive `a`, `b`, and pulse `valid` synchronously on a clock edge
- Only assert `valid` when `busy == 0`
- Sample `z` only when `out_valid == 1`
- Use normal FP32 test values for the main functional checks
- The test requires bit-exact results - off-by-1 errors will fail
- Focus on precise rounding implementation to avoid precision errors

---

## Summary
This module is a 7-cycle, single-issue FP32 multiplier with:
- deterministic handshake timing,
- normal-number focus,
- round-to-nearest-even,
- fixed-latency output valid,
- bit-exact IEEE-754 compliance for normal operands.
