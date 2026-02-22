# File Overview

`m_fixed.h` defines DOOM's 16.16 fixed-point arithmetic type and its associated constants and function prototypes. Because the original DOOM engine was designed to run on 486-era hardware without a floating-point unit (FPU), all geometry, physics, rendering math, and trigonometry in the engine operates on this fixed-point representation rather than floating-point. This header is included virtually everywhere in the codebase.

## Fixed-Point Representation

A `fixed_t` is a 32-bit signed integer interpreted as a binary fixed-point number with 16 bits of integer part and 16 bits of fractional part:

```
bit 31 (sign) | bits 30..16 (integer part) | bits 15..0 (fractional part)
```

The value represented by a raw integer `n` is `n / 65536.0`. Conversely, the real value `v` is stored as `(int)(v * 65536)`.

**Key consequences:**
- The representable range is approximately -32768.0 to +32767.9999847.
- The smallest representable increment is `1/65536 ≈ 0.0000153`.
- `FRACUNIT` (65536, or `1 << 16`) represents the real value 1.0.
- Multiplication of two `fixed_t` values requires a 64-bit intermediate result to avoid overflow; the raw product of two `fixed_t` values would have the decimal point shifted by 32 bits and must be right-shifted by `FRACBITS` (16) to restore it.
- Division requires pre-shifting the dividend left by 16 bits before dividing.

## Global Variables

None. This is a pure interface header.

## Data Structures

### `fixed_t` (typedef)
```c
typedef int fixed_t;
```
A 32-bit signed integer used to represent a 16.16 fixed-point number. Used throughout the engine for positions, velocities, lengths, angles (as fractions of a circle), and scaling factors.

## Preprocessor Constants

| Macro | Value | Meaning |
|---|---|---|
| `FRACBITS` | `16` | Number of fractional bits in a `fixed_t`. The binary point sits 16 bits from the right. |
| `FRACUNIT` | `1 << 16` = 65536 | The `fixed_t` representation of the real value 1.0. Multiply a real integer by `FRACUNIT` to get its `fixed_t` form. |

## Functions

### `FixedMul`
```c
fixed_t FixedMul(fixed_t a, fixed_t b);
```
**Purpose:** Multiply two 16.16 fixed-point numbers and return a 16.16 result.

**Math:** Two `fixed_t` values each have the binary point 16 bits from the right. Their product therefore has the binary point 32 bits from the right (the fractional part doubled). To restore a 16.16 result the product must be right-shifted by 16:

```
result = (a * b) >> 16
```

Because `a * b` can be as large as 32767 * 32767 * 65536^2, the product temporarily needs 64 bits to avoid overflow. The implementation casts both operands to `long long` before multiplying.

**Parameters:**
- `a` - First fixed-point multiplicand.
- `b` - Second fixed-point multiplicand.

**Returns:** `fixed_t` product, `(a * b) >> FRACBITS`.

---

### `FixedDiv`
```c
fixed_t FixedDiv(fixed_t a, fixed_t b);
```
**Purpose:** Divide two 16.16 fixed-point numbers with overflow protection.

**Logic:** Before calling `FixedDiv2`, checks whether the result would overflow a 32-bit integer. The condition `(abs(a) >> 14) >= abs(b)` detects overflow: if the absolute value of `a` shifted right by 14 is at least `|b|`, then `|a/b| * FRACUNIT` would exceed `2^31`. In that case, the function returns `MAXINT` if the signs agree or `MININT` if they differ (saturating division).

**Parameters:**
- `a` - Dividend in 16.16 format.
- `b` - Divisor in 16.16 format.

**Returns:** `fixed_t` quotient, or `MININT`/`MAXINT` on overflow.

---

### `FixedDiv2`
```c
fixed_t FixedDiv2(fixed_t a, fixed_t b);
```
**Purpose:** Perform the actual fixed-point division once overflow is ruled out.

**Math:** To divide `a` by `b` in 16.16 format the result needs to be `(a / b) * FRACUNIT`. The implementation converts both to `double`, divides, multiplies by `FRACUNIT`, then casts back to `fixed_t`. An alternative commented-out implementation shifts `a` left by 16 bits using `long long` arithmetic then divides by `b`.

The function calls `I_Error` if the double result is still out of range (which should not occur when called from `FixedDiv` after the overflow check).

**Parameters:**
- `a` - Dividend (16.16).
- `b` - Divisor (16.16).

**Returns:** `fixed_t` quotient.

---

## Dependencies

- `doomtype.h` - Basic type definitions (`boolean`, etc.)
- `i_system.h` - `I_Error` used in `FixedDiv2` for out-of-range reporting (included in the `.c` file, not this header).

This header itself has no `#include` directives; it relies on the compiler having seen the standard integer types already.
