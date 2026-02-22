# File Overview

`tables.h` is the public header for the DOOM trigonometric lookup table system. It defines the angle type, the BAM (Binary Angle Measurement) constants, the fine-angle system constants, and provides `extern` declarations for the three large precomputed lookup arrays that live in `tables.c`. Nearly every subsystem in DOOM that performs angular or geometric calculations includes this header.

The file also declares the `SlopeDiv` utility function, which is the entry point for the arctangent computation path used throughout the renderer and map code.

---

## Global Variables

### `finesine[5*FINEANGLES/4]`

```c
extern fixed_t finesine[5*FINEANGLES/4];
```

**Effective size:** 10,240 entries.

**Purpose:** Fixed-point sine lookup table. The oversized allocation (10,240 instead of 8,192) allows `finecosine` to alias into the middle of this array; accessing `finesine[i + 2048]` gives `cos(i)` because cosine leads sine by a quarter circle. See `tables.c.md` for the full explanation of this design.

---

### `finecosine`

```c
extern fixed_t* finecosine;
```

**Purpose:** Pointer into the `finesine` array, offset by `FINEANGLES/4 = 2048` entries. Reading `finecosine[i]` is equivalent to reading `finesine[i + 2048]`, yielding the cosine of angle `i`. Initialized in `tables.c` as:

```c
fixed_t* finecosine = &finesine[FINEANGLES/4];
```

This pointer allows code to be written naturally as `finecosine[angle]` without the caller needing to know about the internal offset.

---

### `finetangent[FINEANGLES/2]`

```c
extern fixed_t finetangent[FINEANGLES/2];
```

**Effective size:** 4,096 entries.

**Purpose:** Fixed-point tangent lookup table. Covers angles corresponding to the range (-90, +90) degrees in fixed-point values. Used by the renderer to compute projection slopes for walls and floors.

---

### `tantoangle[SLOPERANGE+1]`

```c
extern angle_t tantoangle[SLOPERANGE+1];
```

**Effective size:** 2,049 entries.

**Purpose:** Arctangent lookup table mapping normalized slope values (0 to `SLOPERANGE` = 2048) to BAM angles in the first octant (0 to 45 degrees). Used in conjunction with `SlopeDiv` to convert 2D direction vectors into BAM angles.

---

## Functions

### `SlopeDiv`

```c
int SlopeDiv(unsigned num, unsigned den);
```

**Purpose:** Converts a slope (expressed as a ratio `num/den`) into a normalized index in the range [0, SLOPERANGE] for use with `tantoangle[]`. Acts as the first half of a fast arctangent operation. See `tables.c.md` for full documentation.

---

## Data Structures

### `angle_t`

```c
typedef unsigned angle_t;
```

**Purpose:** Represents a direction angle in BAM (Binary Angle Measurement) format. An unsigned 32-bit integer where the full range 0x00000000 through 0xFFFFFFFF represents exactly one full rotation (360 degrees). Wraparound is intentional and handled correctly by unsigned arithmetic.

**Advantages of BAM:**
- No special-casing needed for angle wraparound (e.g., from 359 degrees to 0 degrees); unsigned overflow handles it naturally.
- Angle addition, subtraction, and comparison all work correctly using standard arithmetic.
- Can represent any direction with a precision of approximately 8.4e-8 degrees (360 / 2^32).

**Common angle_t values:**

| Symbolic name | Hex value | Degrees |
|--------------|-----------|---------|
| `ANG45` | `0x20000000` | 45 |
| `ANG90` | `0x40000000` | 90 |
| `ANG180` | `0x80000000` | 180 |
| `ANG270` | `0xC0000000` | 270 |

---

## Macro Constants

### Fine Angle System

| Macro | Value | Purpose |
|-------|-------|---------|
| `FINEANGLES` | `8192` | Number of discrete fine angles in a full circle. Divides the BAM circle into 8192 equal parts. |
| `FINEMASK` | `8191` (`FINEANGLES - 1`) | Bitmask used to wrap a fine angle index into the valid range, equivalent to `% FINEANGLES`. |
| `ANGLETOFINESHIFT` | `19` | Right-shift amount to convert a BAM angle to a fine angle index: `fine = bam >> 19`. Equivalently, BAM has 2^32 units/circle, fine has 2^13, and 32-13 = 19. |

**Conversion example:**
```c
// Get the fine-angle index from a BAM angle
int fine_idx = (bam_angle >> ANGLETOFINESHIFT) & FINEMASK;
// Look up sine
fixed_t s = finesine[fine_idx];
// Look up cosine
fixed_t c = finecosine[fine_idx];
```

### BAM Angle Constants

| Macro | Value | Purpose |
|-------|-------|---------|
| `ANG45` | `0x20000000` | 45-degree BAM angle constant. |
| `ANG90` | `0x40000000` | 90-degree BAM angle constant. East direction. |
| `ANG180` | `0x80000000` | 180-degree BAM angle constant. West direction. |
| `ANG270` | `0xC0000000` | 270-degree BAM angle constant. South direction. |

### Slope / Arctangent System

| Macro | Value | Purpose |
|-------|-------|---------|
| `SLOPERANGE` | `2048` | Maximum normalized slope index. Represents a 45-degree slope (rise equals run). |
| `SLOPEBITS` | `11` | Number of bits in the slope range: `2^11 = 2048`. |
| `DBITS` | `5` | `FRACBITS - SLOPEBITS = 16 - 11 = 5`. Used in fast division logic in the renderer. |

### Platform-Specific

| Macro | Condition | Value | Purpose |
|-------|-----------|-------|---------|
| `PI` | Non-Linux builds only | `3.141592657` | Pi constant for systems without `<math.h>`. On Linux, `<math.h>` is included instead. |

---

## Dependencies

| File | Reason |
|------|--------|
| `m_fixed.h` | Provides `fixed_t` (the 16.16 fixed-point type used for table values) and `FRACBITS` (16, used in the `DBITS` macro). |
| `<math.h>` | Included on Linux builds for the system `M_PI` constant. Not needed for the tables themselves but available for any floating-point math in the build system. |
