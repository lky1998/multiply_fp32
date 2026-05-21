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

CRITICAL EXPONENT RANGES:
- exp = 0: Zero or subnormal (handle as zero for simplicity)
- exp = 1-254: Normal numbers
- exp = 255: Infinity or NaN (handle as infinity for simplicity)

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

### CRITICAL: Handle Edge Cases
While normal numbers are the focus, you MUST handle these cases correctly:
- **Underflow**: If final exponent < 1, result = 0x00000000 (positive zero)
- **Overflow**: If final exponent > 254, result = 0x7F800000 (positive infinity) or 0xFF800000 (negative infinity)
- **Zero inputs**: If either input has exp=0, result = 0x00000000 (zero)

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
2. Special case detection (zero, underflow, overflow)
3. Normal-number sign handling
4. Exponent extraction and unbiased conversion
5. 24-bit mantissa formation with hidden 1
6. Mantissa multiplication (48-bit precision)
7. Normalization
8. Round-to-nearest-even
9. Pack result with overflow/underflow checks

---

## FSM / Pipeline Stages

The design uses a 7-stage internal sequence controlled by `busy` and `counter`.

### Stage 1 — Register inputs
- Sample `a` and `b`
- Capture sign, exponent, and fraction fields
- Initialize internal registers

### Stage 2 — Special case detection and mantissa building
- Check for zero inputs: if a_exp==0 or b_exp==0, set result to zero
- For normal operands, form 24-bit mantissas as `{1'b1, frac}`
- Convert biased exponent to unbiased exponent by subtracting 127

### Stage 3 — Compute result sign and core multiply setup
- Compute result sign: `z_s = a_s ^ b_s`
- Prepare exponent sum and mantissa multiplication
- Early exit for zero cases

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

### Stage 7 — Pack result with overflow/underflow handling
- CRITICAL: Check for underflow/overflow BEFORE packing
- If final_exp < 1: result = {z_s, 8'h00, 23'h000000} (signed zero)
- If final_exp > 254: result = {z_s, 8'hFF, 23'h000000} (signed infinity)
- Otherwise: result = {z_s, (final_exp + 127)[7:0], final_mantissa}
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

### Normalization Logic
After multiplication, check if product[47] == 1:
- If yes: result ≥ 2.0, shift right by 1, increment exponent
- If no: result < 2.0, no shift needed

Extract rounding bits accordingly:
- Normalized: mantissa=product[46:24], guard=product[23], round=product[22], sticky=|product[21:0]
- Not normalized: mantissa=product[45:23], guard=product[22], round=product[21], sticky=|product[20:0]

---

## CRITICAL: Overflow and Underflow Handling

### Underflow Detection
After normalization and rounding, check final exponent: if (final_exp_adjusted < 1) result = {result_sign, 8'h00, 23'h000000}

### Overflow Detection  
After normalization and rounding, check final exponent: if (final_exp_adjusted > 254) result = {result_sign, 8'hFF, 23'h000000}

### Zero Input Handling
In stage 2, check for zero inputs: if (a_exp == 8'h00 || b_exp == 8'h00) set result_is_zero flag and skip normal computation

### Implementation Priority
1. **First implement normal number path** (stages 1-7)
2. **Then add underflow/overflow checks** in stage 7
3. **Finally add zero input detection** in stage 2

---

## Normal-Number Behavior
For normal inputs:
- Hidden leading 1 is always present in the mantissa
- The result should follow standard FP32 multiplication rules
- RNE must be applied to the final normalized mantissa
- Sign is XOR of input signs

---

## Special-Case Policy
**IMPORTANT**: While normal numbers are the primary focus, the test cases include edge cases that cause underflow/overflow. You MUST handle:
- Zero inputs (exp=0) → result is zero
- Underflow (final exp < 1) → result is zero  
- Overflow (final exp > 254) → result is infinity

These cases are simple to implement and will significantly improve your pass rate.

---

## Verification Notes
Recommended testbench usage:
- Drive `a`, `b`, and pulse `valid` synchronously on a clock edge
- Only assert `valid` when `busy == 0`
- Sample `z` only when `out_valid == 1`
- Use normal FP32 test values for the main functional checks
- The test requires bit-exact results - off-by-1 errors will fail
- Focus on precise rounding implementation to avoid precision errors
- Handle underflow/overflow cases correctly to avoid major errors

---

## Summary
This module is a 7-cycle, single-issue FP32 multiplier with:
- deterministic handshake timing,
- normal-number focus with essential edge case handling,
- round-to-nearest-even,
- fixed-latency output valid,
- bit-exact IEEE-754 compliance for normal operands,
- correct underflow/overflow behavior for edge cases.
