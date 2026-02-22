# File Overview

**File:** `linuxdoom-1.10/m_swap.c`
**Module prefix:** `Swap` (function names), `SHORT` / `LONG` (macros defined in the header)

`m_swap.c` provides byte-order (endianness) conversion routines for 16-bit and 32-bit integers. DOOM's WAD file format stores all multi-byte integers in **little-endian** byte order (least significant byte first), which is native on the x86 processors that DOOM was originally designed for. On big-endian architectures (such as SPARC, MIPS, or PowerPC) the bytes must be swapped before use.

The functions in this file are only compiled on big-endian targets (`#ifndef __BIG_ENDIAN__` guards them out on little-endian builds). On little-endian systems the `SHORT()` and `LONG()` macros defined in `m_swap.h` expand to identity operations (no-ops), so there is zero runtime cost for the vast majority of platforms.

The source file note in the `rcsid` string mistakenly identifies the file as `m_bbox.c` - this is a copy-paste error in the original id Software source and does not affect functionality.

---

## Global Variables

None. This file contains only function definitions (conditionally compiled) and the `rcsid` version string.

---

## Functions

Both functions are only defined when `__BIG_ENDIAN__` is **not** defined. On a little-endian system the translation unit contributes no code at all (the functions are replaced by identity macros in the header).

---

### `SwapSHORT`

```c
unsigned short SwapSHORT(unsigned short x)
```

**Purpose:** Reverse the byte order of a 16-bit unsigned integer, converting between little-endian (WAD format) and the native big-endian representation.

**Parameters:**
- `x` - the 16-bit value to byte-swap.

**Returns:** The byte-swapped 16-bit value.

**Key logic:**
```c
return (x >> 8) | (x << 8);
```

- `x >> 8` shifts the high byte down to the low byte position.
- `x << 8` shifts the low byte up to the high byte position.
- Bitwise OR combines them.

On a 16-bit value `0xAABB`, this produces `0xBBAA`. No masking is needed because both shift results are naturally zero in their unused halves for a 16-bit value (the comment in the source acknowledges this).

**Used via:** The `SHORT(x)` macro in `m_swap.h`, which calls `SwapSHORT` only on big-endian builds and expands to `(x)` on little-endian builds.

---

### `SwapLONG`

```c
unsigned long SwapLONG(unsigned long x)
```

**Purpose:** Reverse the byte order of a 32-bit unsigned integer, converting between little-endian (WAD format) and the native big-endian representation.

**Parameters:**
- `x` - the 32-bit value to byte-swap.

**Returns:** The byte-swapped 32-bit value.

**Key logic:**
```c
return
    (x >> 24)
    | ((x >> 8)  & 0xff00)
    | ((x << 8)  & 0xff0000)
    | (x << 24);
```

For a 32-bit value `0xAABBCCDD` this produces `0xDDCCBBAA`:

| Expression | Contribution |
|------------|-------------|
| `x >> 24` | Moves byte 3 (`0xAA`) to byte 0 position |
| `(x >> 8) & 0xff00` | Moves byte 2 (`0xBB`) to byte 1 position |
| `(x << 8) & 0xff0000` | Moves byte 1 (`0xCC`) to byte 2 position |
| `x << 24` | Moves byte 0 (`0xDD`) to byte 3 position |

Masking with `0xff00` and `0xff0000` is required here (unlike in `SwapSHORT`) to prevent the shifted values from overlapping in adjacent byte positions.

**Used via:** The `LONG(x)` macro in `m_swap.h`, which calls `SwapLONG` only on big-endian builds.

---

## Data Structures

None.

---

## Relationship with `m_swap.h`

The `.h` file defines the `SHORT()` and `LONG()` macros that all engine code uses:

```c
// big-endian build:
#define SHORT(x)   ((short)SwapSHORT((unsigned short)(x)))
#define LONG(x)    ((long)SwapLONG((unsigned long)(x)))

// little-endian build:
#define SHORT(x)   (x)
#define LONG(x)    (x)
```

This design means that all WAD-data access code is written portably as `SHORT(field)` and `LONG(field)`, but on x86 (the common case) the preprocessor eliminates all calls to this file entirely. `m_swap.c` only matters when porting DOOM to a big-endian platform.

---

## Dependencies

| Module | Header | What is used |
|--------|--------|-------------|
| Self header | `m_swap.h` | Provides the `#pragma implementation` linkage for G++ and the `__BIG_ENDIAN__` conditional |

No other dependencies. This is an intentionally minimal, standalone utility.
