# File Overview

**File:** `linuxdoom-1.10/m_swap.h`
**Module prefix:** `Swap` (functions), `SHORT` / `LONG` (macros)

This header defines DOOM's portability layer for byte-order conversion. All multi-byte integers stored in WAD files (lump sizes, offsets, map geometry coordinates, texture dimensions, etc.) use **little-endian** byte order. On little-endian hardware (x86) no conversion is needed; on big-endian hardware (SPARC, MIPS, PowerPC) bytes must be swapped before the values are used.

The header provides two macros - `SHORT()` and `LONG()` - that every piece of WAD-reading and binary-format code uses. On little-endian builds these macros are pure identity operations that the compiler optimises away entirely. On big-endian builds they call the swap functions defined in `m_swap.c`.

Engine code should always access raw binary data through these macros, never via direct casts, to ensure portability.

---

## Global Variables

None.

---

## Functions

The function declarations appear only when `__BIG_ENDIAN__` is defined. On little-endian systems the macros bypass the functions entirely.

### `SwapSHORT` (big-endian only)

```c
short SwapSHORT(short);
```

**Purpose:** Swap the two bytes of a 16-bit integer, converting between WAD little-endian storage and the native big-endian CPU format.

**Parameters:** A 16-bit integer value.

**Returns:** The byte-swapped 16-bit value.

Implemented in `m_swap.c`. Not called directly by engine code; invoked only through the `SHORT()` macro.

---

### `SwapLONG` (big-endian only)

```c
long SwapLONG(long);
```

**Purpose:** Reverse the four bytes of a 32-bit integer, converting between WAD little-endian storage and the native big-endian CPU format.

**Parameters:** A 32-bit integer value.

**Returns:** The byte-swapped 32-bit value.

Implemented in `m_swap.c`. Not called directly by engine code; invoked only through the `LONG()` macro.

---

## Macros

### `SHORT(x)`

```c
// Big-endian:
#define SHORT(x)   ((short)SwapSHORT((unsigned short)(x)))

// Little-endian:
#define SHORT(x)   (x)
```

**Purpose:** Convert a 16-bit WAD value to native byte order. Used whenever reading any `short` field from a WAD lump structure or PCX header: map vertex coordinates, linedef/sidedef indices, texture dimensions, patch offsets, etc.

**Usage pattern throughout the engine:**
```c
v->x = SHORT(ml->x);           // Map vertex X coordinate
p->width = SHORT(patch->width); // Patch width from graphic lump
pcx->xmax = SHORT(width - 1);   // PCX header field
```

On little-endian (x86), the macro expands to just `(x)` - the compiler sees it as a no-op cast and generates no extra instructions.

---

### `LONG(x)`

```c
// Big-endian:
#define LONG(x)    ((long)SwapLONG((unsigned long)(x)))

// Little-endian:
#define LONG(x)    (x)
```

**Purpose:** Convert a 32-bit WAD value to native byte order. Used when reading 32-bit fields from WAD structures: WAD header magic numbers, lump offsets, lump sizes, and any 32-bit binary data fields.

**Usage pattern throughout the engine:**
```c
header->numlumps = LONG(header->numlumps);  // WAD directory entry count
header->infotableofs = LONG(header->infotableofs); // WAD directory offset
```

---

## Data Structures

None defined in this header.

---

## Compilation Notes

The header uses a G++ pragma for explicit template instantiation linkage:

```c
#ifdef __GNUG__
#pragma interface
#endif
```

This corresponds to `#pragma implementation "m_swap.h"` in `m_swap.c` and is a GNU C++ extension used in the era when this code was written to control where template or inline function definitions are emitted. It has no effect on plain C compilation.

---

## Dependencies

None. This header is entirely self-contained and has no `#include` dependencies. It is designed to be included very early in the include chain by other headers.

---

## Usage Scope

`SHORT()` and `LONG()` are used pervasively throughout the engine wherever binary data is read from WAD lumps or binary file formats:

| Module | Context |
|--------|---------|
| `w_wad.c` | WAD header fields, lump directory offsets and sizes |
| `r_data.c` | Texture and patch header fields |
| `p_setup.c` | All map lump structures (vertices, linedefs, sidedefs, sectors, nodes, segs, blockmap, reject) |
| `m_misc.c` | PCX screenshot header fields |
| `v_video.c` | Patch graphic header fields |
| `r_things.c` | Sprite frame patch dimensions |
