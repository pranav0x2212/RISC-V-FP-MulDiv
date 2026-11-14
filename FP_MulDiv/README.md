# FP Multiplier / Divider

One module, `fp_muldiv`: IEEE-754 **binary32** (single precision) `a * b` or `a / b`. CI checks it with a trusted harness; the expected
results come from an exact model, so "correct" means *bit-exact*. (Implement the multiplier and the divider inside this one module.)

### Module Specification

```SystemVerilog
module fp_muldiv (
  input  logic        clk,
  input  logic        reset,      // active high
  input  logic        in_valid,   // high for ONE clock edge; div, a, b are valid only then
  input  logic        div,        // 0: a * b, 1: a / b
  input  logic [31:0] a,
  input  logic [31:0] b,
  output logic        out_valid,  // high while result is valid (same cycle .. 64 cycles after in_valid)
  output logic [31:0] result
);
```

### Ports (names, widths and directions are fixed)

| Port | Dir | Width | Meaning |
|---|---|---|---|
| `clk` | in | 1 | clock, rising edge |
| `reset` | in | 1 | active high; synchronous or asynchronous, you choose. Held for several cycles before the first operation |
| `in_valid` | in | 1 | high for **one** clock edge when an operation is presented |
| `div` | in | 1 | `0`: `a * b`, `1`: `a / b` |
| `a`, `b` | in | 32 | binary32 operands, valid **only** in the cycle `in_valid` is high |
| `out_valid` | out | 1 | high while `result` is valid |
| `result` | out | 32 | binary32 result |

### Handshake (latency-insensitive)

* The testbench drives the operand(s), `div` with `in_valid = 1` across exactly one rising clock edge. In the next cycle those
  inputs are changed to garbage, so **capture them on that edge** (register them, or finish the computation combinationally in that
  cycle).
* The result is the value of `result` in the **first cycle `out_valid` is high**: the same cycle as `in_valid` (a purely
  combinational design with `out_valid = in_valid`), the next cycle, or up to **64 cycles** later (iterative dividers / square-root
  units). Fixed or variable latency are both fine.
* `out_valid` must be low whenever no result is being presented. A one-cycle pulse is the simplest; do not hold it high.
* One operation at a time: the testbench waits for `out_valid` and then idles two cycles before the next operation.
* No result within 64 cycles counts as a wrong result.

### Arithmetic

* **Format.** `[31]` sign, `[30:23]` biased exponent (bias 127), `[22:0]` fraction. Exponent `0` with fraction `0` is a zero, exponent `0`
  with a non-zero fraction is a **subnormal** (value `0.fraction * 2^-126`, no hidden bit). Exponent `255` is infinity (fraction `0`) or
  NaN (fraction `!= 0`).
* **Rounding.** Round to nearest, ties to even, on every result (including subnormal results). **Subnormals** are supported on inputs
  and outputs: no flush-to-zero. Overflow rounds to infinity with the correct sign.
* **NaN.** Any NaN operand (quiet or signalling, any sign or payload) and every invalid operation give the **canonical quiet NaN
  `32'h7FC0_0000`**. Exception flags are not part of the interface.
* **Sign.** The sign of the result is `sign(a) XOR sign(b)`, also for zero results, infinities, and results that underflow to zero.
* **Multiply.** `inf * finite` (non-zero) = infinity; `inf * 0` is invalid (NaN); `0 * finite` = signed zero.
* **Divide.** `x / 0` (x finite non-zero or infinity) = infinity with the XOR sign; `0 / 0` and `inf / inf` are invalid (NaN);
  `finite / inf` = signed zero; `0 / finite` = signed zero; `inf / finite` = infinity.

### What is checked

About 15,000 operations: every pair of 24 corner values (zeros, subnormals, min/max normal, infinities, NaNs) with both operations,
then constrained-random operands (random bit patterns, mid-range exponents, subnormal operands, overflow and underflow ranges, short
mantissas giving exact results, equal operands, powers of two, results near the subnormal boundary). Every result must match bit for bit.

### Hints

* Multiply: the product of two 24-bit significands is 48 bits; keep guard/round/sticky bits and normalise with a leading-one search.
* Divide: normalise subnormal operands first (shift the significand up, lower the exponent), produce at least 26 quotient bits and
  use the non-zero remainder as a sticky bit. An iterative (restoring / non-restoring) divider is fine: it just takes more cycles.
* A rounding carry (`1.111...1` becoming `10.000...0`) increments the exponent and can overflow to infinity; a result below the
  smallest normal must be rounded at the **subnormal** position, not at 24 bits.
* Yosys 0.33 (the CI synthesizer) cannot parse assignment patterns (`'{default: 0}`), and `small` is a reserved word.

### Interface

> [!IMPORTANT]
> Your module, port and parameter names are up to you. Your design must have the same number of ports, with the same directions and widths, as shown above, and must behave as specified. Map your names to this spec in `rnr-stage.yaml` (see the assignment handout).
