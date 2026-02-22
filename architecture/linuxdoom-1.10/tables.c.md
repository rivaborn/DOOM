# File Overview

`tables.c` is one of the most fundamental files in the DOOM engine. It contains three enormous precomputed lookup tables used for all trigonometric math in the game, plus the `SlopeDiv` utility function. Because DOOM was designed to run on 386/486 CPUs without floating-point hardware (or with slow FPUs), all geometry, rendering, and physics calculations use integer fixed-point arithmetic. Computing sine, cosine, and tangent at runtime would have been prohibitively slow, so id Software precomputed all needed values at build time and stored them in these tables.

The file is almost entirely data: roughly 2,100 lines of comma-separated integer literals forming three arrays. Only the 12-line `SlopeDiv` function contains actual executable logic.

## The FINEANGLES System

DOOM uses a custom angle representation called **BAM (Binary Angle Measurement)**. A full circle is represented by the full range of a 32-bit unsigned integer (0x00000000 to 0xFFFFFFFF). This means there are 2^32 (approximately 4.3 billion) possible angles in a full rotation, which gives extremely fine angular precision with no discontinuity at the wraparound point.

For the lookup tables, a coarser resolution of **8192 fine angles** (FINEANGLES = 8192) is used. This is 1/512th of the full BAM range. To convert from a BAM angle to a fine angle index, the engine shifts right by **ANGLETOFINESHIFT = 19** bits:

```
fine_index = bam_angle >> 19
```

This works because: 2^32 / 2^19 = 2^13 = 8192 = FINEANGLES. Each fine angle unit therefore covers approximately 0.044 degrees (360 / 8192).

**Why 8192?** This number balances table size against precision. At 8192 entries, the sine table occupies 8192 * 4 = 32,768 bytes (32 KB). The tangent table is half that at 4096 entries. Together with the arctangent table (2049 entries), the total is about 58 KB of read-only data — acceptable for a game targeting 4 MB of RAM.

### BAM Constants

The following BAM angle constants are defined in `tables.h`:

| Constant | Hex Value | Degrees |
|----------|-----------|---------|
| `ANG45` | `0x20000000` | 45 degrees |
| `ANG90` | `0x40000000` | 90 degrees |
| `ANG180` | `0x80000000` | 180 degrees |
| `ANG270` | `0xC0000000` | 270 degrees |

---

## Global Variables

### `finetangent[4096]`

```c
int finetangent[4096];
```

**Type:** Array of 4096 `int` (treated as `fixed_t`, i.e., 16.16 fixed-point).

**Purpose:** Tangent lookup table. Covers angles from -90 degrees (exclusive) to +90 degrees (exclusive), i.e., the full range of tangent values. Index 0 corresponds to approximately -89.96 degrees and index 4095 corresponds to approximately +89.96 degrees, passing through 0 at index 2048.

**Usage:** Used primarily in `r_main.c` during view frustum setup and in `p_map.c` for line-of-sight calculations. When the renderer needs `tan(angle)`, it looks up `finetangent[angle >> ANGLETOFINESHIFT]` (with appropriate range clamping, since tangent covers only half the fine-angle range).

**Value range:** The values are scaled fixed-point numbers. Extreme values near ±90 degrees reach approximately ±170,910,304 in fixed-point, corresponding to very large tangent values. A tangent of 1.0 (45 degrees) would be represented as `FRACUNIT = 65536`.

---

### `finesine[10240]`

```c
int finesine[10240];   // declared as fixed_t finesine[5*FINEANGLES/4]
```

**Type:** Array of 10,240 `int` (as `fixed_t`).

**Purpose:** Sine lookup table. The array is logically sized for **10,240 entries** (5/4 of 8192), which is 8192 entries for a full circle **plus an extra 2048 entries**. The extra quarter-circle of entries allows the cosine function to reuse this same array via a pointer offset.

**Usage:** Indexed by fine angle (0 to 8191 for a full circle). `finesine[0]` = 0, `finesine[2048]` = FRACUNIT (1.0), `finesine[4096]` = 0, `finesine[6144]` = -FRACUNIT (-1.0). Values are FRACUNIT-scaled fixed-point where FRACUNIT = 65536 represents 1.0.

**The cosine trick:** The cosine of an angle is the sine of (angle + 90 degrees). In fine-angle units, 90 degrees = FINEANGLES/4 = 2048. So `finecosine` is declared as:

```c
fixed_t* finecosine = &finesine[FINEANGLES/4];   // = &finesine[2048]
```

When you access `finecosine[i]`, you are actually reading `finesine[i + 2048]`, which is `sin(angle + 90) = cos(angle)`. This clever aliasing is why the array has 10,240 entries instead of 8,192 — the extra 2048 entries at the end prevent an out-of-bounds access when cosine indexes near the end of the circle.

---

### `tantoangle[2049]`

```c
angle_t tantoangle[2049];   // declared as tantoangle[SLOPERANGE+1]
```

**Type:** Array of 2,049 `unsigned` (as `angle_t`, BAM angles).

**Purpose:** Arctangent lookup table. Given a tangent value (as a ratio), returns the BAM angle. This is the inverse of `finetangent`.

**The SLOPERANGE System:** The arctangent table uses a different coordinate system from the fine-angle tables. The slope (rise/run) is expressed as an integer ratio where the range 0 to `SLOPERANGE` (= 2048) represents slopes from 0 to 1 (i.e., angles from 0 to 45 degrees). To compute an arbitrary angle from a vector (dx, dy), the engine uses `SlopeDiv(dy, dx)` to normalize the slope into the 0-2048 range, then looks up `tantoangle[result]` to get the 0-to-45-degree angle, and then uses the signs of dx and dy to determine the actual quadrant.

**The +1 size:** The table has 2049 entries rather than 2048 so that the degenerate case of slope = 1 (exactly 45 degrees, when dx == dy) can be handled without an additional bounds check.

---

## Functions

### `SlopeDiv`

```c
int SlopeDiv(unsigned num, unsigned den);
```

**Purpose:** Computes a normalized slope value in the range [0, SLOPERANGE] from a numerator and denominator. Used as the first step in converting a 2D vector into an angle.

**Parameters:**
- `num` - The numerator of the slope (typically the absolute value of the y-component of a direction vector, i.e., the "rise").
- `den` - The denominator of the slope (typically the absolute value of the x-component, i.e., the "run").

**Return value:** An integer in the range [0, 2048] (SLOPERANGE) representing the normalized slope, suitable as an index into `tantoangle[]`. Returns `SLOPERANGE` (the maximum value, representing 45 degrees) if `den` is very small (less than 512), to avoid division by zero.

**Key logic:**

```c
if (den < 512)
    return SLOPERANGE;
ans = (num << 3) / (den >> 8);
return ans <= SLOPERANGE ? ans : SLOPERANGE;
```

The calculation `(num << 3) / (den >> 8)` is equivalent to `(num * 8) / (den / 256)` = `num * 2048 / den`. This scales the ratio `num/den` to the range [0, 2048] when num <= den (i.e., when the slope is at most 1 / 45 degrees). The caller must ensure that `num <= den` before calling (by swapping if necessary), so the result is always in the correct half-angle range before the quadrant adjustment.

**Called by:** `R_PointToAngle` and `R_PointToAngle2` in `r_main.c`, which use this to convert world-space vectors to BAM angles for rendering.

---

## Data Structures

The file defines no structs or typedefs. The `angle_t` typedef (as `unsigned`) is defined in `tables.h`, not here.

---

## Dependencies

| File | Reason |
|------|--------|
| `tables.h` | Provides all type definitions (`fixed_t`, `angle_t`), macro constants (`FINEANGLES`, `SLOPERANGE`, etc.), and the `extern` declarations for the three tables. |
| `m_fixed.h` | Transitively required through `tables.h` for the `fixed_t` type and `FRACBITS`/`FRACUNIT` constants. |

The file has no other dependencies. It is a pure data/utility module with no dependencies on game state, rendering, or I/O.
