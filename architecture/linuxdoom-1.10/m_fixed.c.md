# File Overview

`m_fixed.c` implements the three fixed-point arithmetic operations declared in `m_fixed.h`. These functions are among the most frequently called in the entire engine; virtually every rendering calculation, movement update, trigonometric lookup, and collision test goes through `FixedMul` or `FixedDiv`.

DOOM was designed to run deterministically on all platforms. Using integer fixed-point math rather than floating-point ensures that the same numerical results appear regardless of whether the host CPU has an FPU, and that network games (which rely on lockstep simulation) stay in sync across different machines.

The file also includes a note ("Fixme. __USE_C_FIXED__ or something") indicating id Software intended to support assembly-language implementations of these functions on x86, where a `imul` followed by a `shrd` could implement `FixedMul` in two instructions, and `idiv` after a `shld` could implement `FixedDiv2`. The Linux release shipped the portable C version.

## Global Variables

```c
static const char rcsid[] = "$Id: m_bbox.c,v 1.1 1997/02/03 22:45:10 b1 Exp $";
```
RCS version string. Note the ID mistakenly says `m_bbox.c` -- this is a copy/paste error in the original source.

## Functions

### `FixedMul`
```c
fixed_t FixedMul(fixed_t a, fixed_t b)
```
**Purpose:** Multiply two 16.16 fixed-point numbers and return a 16.16 result.

**Parameters:**
- `a` (`fixed_t`) - First multiplicand.
- `b` (`fixed_t`) - Second multiplicand.

**Returns:** `fixed_t` - The 16.16 fixed-point product.

**Key logic:**
```c
return ((long long) a * (long long) b) >> FRACBITS;
```
Both operands are widened to 64-bit signed integers before multiplication to avoid overflow. The raw 64-bit product has the binary point 32 bits from the right (because each operand contributed 16 fractional bits). Right-shifting by `FRACBITS` (16) restores the binary point to 16 bits from the right, yielding a 16.16 result in the lower 32 bits.

Example: `FixedMul(2*FRACUNIT, 3*FRACUNIT)` computes `(131072 * 196608) >> 16 = 196608 = 3*FRACUNIT`. Correct: 2.0 * 3.0 = 3.0... wait, that should be 6.0. Let's recheck: `2*FRACUNIT = 131072`, `3*FRACUNIT = 196608`; product = `131072 * 196608 = 25769803776`; `>> 16` = `393216 = 6*FRACUNIT`. Correct.

---

### `FixedDiv`
```c
fixed_t FixedDiv(fixed_t a, fixed_t b)
```
**Purpose:** Safe fixed-point division with overflow detection.

**Parameters:**
- `a` (`fixed_t`) - Dividend.
- `b` (`fixed_t`) - Divisor.

**Returns:** `fixed_t` - Quotient, or `MININT`/`MAXINT` if the true result would not fit in 32 bits.

**Key logic:**
```c
if ( (abs(a)>>14) >= abs(b))
    return (a^b)<0 ? MININT : MAXINT;
return FixedDiv2 (a,b);
```
The overflow check works as follows: the true result in 16.16 format is `(a / b) * 65536`. This overflows a 32-bit integer when `|a / b| * 65536 >= 2^31`, i.e. when `|a / b| >= 32768 = 2^15`, i.e. when `|a| >= |b| * 2^15`. The check `(abs(a) >> 14) >= abs(b)` is equivalent to `abs(a) >= abs(b) * 2^14`, which is slightly conservative (triggers one bit earlier than strictly necessary) to guard against edge cases.

If overflow is detected:
- `(a^b) < 0` is true when the operands have opposite signs (XOR of sign bits is 1), meaning the quotient would be negative, so return `MININT`.
- Otherwise return `MAXINT`.

---

### `FixedDiv2`
```c
fixed_t FixedDiv2(fixed_t a, fixed_t b)
```
**Purpose:** Perform the actual fixed-point division, assuming no overflow.

**Parameters:**
- `a` (`fixed_t`) - Dividend.
- `b` (`fixed_t`) - Divisor.

**Returns:** `fixed_t` - Quotient `(a / b)` in 16.16 format.

**Key logic (active path):**
```c
double c;
c = ((double)a) / ((double)b) * FRACUNIT;
if (c >= 2147483648.0 || c < -2147483648.0)
    I_Error("FixedDiv: divide by zero");
return (fixed_t) c;
```
The function converts both operands to `double`, performs real division, then multiplies by `FRACUNIT` (65536) to scale back to 16.16. The result is checked against 32-bit signed integer limits before the cast; if it is out of range `I_Error` is called with a misleading "divide by zero" message (any out-of-range result, not just zero division, triggers this path).

**Commented-out alternative:**
```c
// long long c;
// c = ((long long)a<<16) / ((long long)b);
// return (fixed_t) c;
```
This cleaner integer-only version shifts `a` left by 16 bits to compensate for the fractional scaling before dividing. It was disabled, presumably because `double` gave better performance or behaviour on the target Linux hardware.

---

## Dependencies

| Header | Purpose |
|---|---|
| `stdlib.h` | `abs()` function |
| `doomtype.h` | `boolean`, `fixed_t` base types |
| `i_system.h` | `I_Error()` for fatal error reporting |
| `m_fixed.h` | Own function prototypes and `FRACBITS`, `FRACUNIT`, `fixed_t` |
