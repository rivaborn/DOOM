# File Overview

`doomtype.h` defines the most fundamental types and numeric limits used throughout the DOOM engine. It is designed to be a minimal, portable foundation header that can be included anywhere without pulling in heavier dependencies. Its primary job is to define `boolean` and `byte` — two types that DOOM uses pervasively but which are not guaranteed to exist in standard C89 — and to provide portable numeric limit macros when the system's `<values.h>` is not available.

This file is included (directly or transitively) by virtually every other header in the engine.

## Global Variables

This header defines no global variables.

## Functions

This header declares no functions.

## Data Structures / Typedefs

### `boolean`

```c
#ifdef __cplusplus
typedef bool boolean;
#else
typedef enum {false, true} boolean;
#endif
```

A portable boolean type. In C++, it maps to the built-in `bool`. In C, it is an enum with values `false` (0) and `true` (1). This ensures consistent semantics across both compilation modes, since the codebase was designed to compile as either C or C++.

**Note:** The guard macro `__BYTEBOOL__` prevents double-definition when multiple headers that define `boolean` and `byte` are included together.

---

### `byte`

```c
typedef unsigned char byte;
```

An 8-bit unsigned integer. Used extensively for raw pixel data, WAD lump content, and as the element type of screen buffers. `byte` is more readable and semantically clear than `unsigned char` throughout the codebase.

## Numeric Limit Constants

When the `LINUX` macro is defined, `<values.h>` is included instead (which provides system-native limit macros). Otherwise, the following are defined:

| Constant | Value | Type | Meaning |
|----------|-------|------|---------|
| `MAXCHAR` | `0x7f` | `char` | Maximum signed 8-bit value (127). |
| `MAXSHORT` | `0x7fff` | `short` | Maximum signed 16-bit value (32767). |
| `MAXINT` | `0x7fffffff` | `int` | Maximum positive 32-bit signed integer. |
| `MAXLONG` | `0x7fffffff` | `long` | Maximum positive 32-bit long value. |
| `MINCHAR` | `0x80` | `char` | Minimum signed 8-bit value (-128, as two's complement). |
| `MINSHORT` | `0x8000` | `short` | Minimum signed 16-bit value (-32768). |
| `MININT` | `0x80000000` | `int` | Maximum negative 32-bit integer (-2147483648). |
| `MINLONG` | `0x80000000` | `long` | Maximum negative 32-bit long value. |

These limits are used throughout the engine wherever bounds checks or sentinel values are needed — most notably, `MAXINT` is used in `TryRunTics` to initialize `lowtic` before a minimum-finding loop.

## Dependencies

This header has no `#include` dependencies of its own (aside from the conditional `<values.h>` on Linux). It is entirely self-contained, which is what allows it to serve as the foundational type header for the rest of the engine.
