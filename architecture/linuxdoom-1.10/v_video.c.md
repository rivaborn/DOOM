# File Overview

`v_video.c` implements DOOM's primary 2D screen-space rendering layer. It is the abstraction layer between all higher-level drawing code (status bar, intermission screens, menus, patches) and the raw framebuffer memory. The module manages five software screen buffers, provides blitting and patch-drawing routines, and defines the gamma correction lookup tables.

DOOM renders in software to 320x200 pixel linear framebuffers. `v_video.c` provides the functions to write to those buffers without any knowledge of the actual display hardware — hardware interaction is handled separately by `i_video.c`. Screen 0 is the primary display buffer that `I_UpdateScreen` flips to the display each frame. Screens 1-4 are auxiliary buffers used for double-buffering, wipe effects, saving backgrounds, and other intermediate compositing.

The **patch format** handled here is DOOM's native column-based RLE sprite/graphic format. Each patch is a sparse image divided into vertical columns, with each column broken into opaque "posts" separated by transparent gaps. This format enables alpha-masked drawing without a separate mask channel.

---

## Global Variables

### `screens[5]`

```c
byte* screens[5];
```

**Type:** Array of 5 pointers to byte arrays.

**Purpose:** The five software framebuffers. Each valid screen pointer references a region of `SCREENWIDTH * SCREENHEIGHT` bytes (320 * 200 = 64,000 bytes at standard resolution). Only screens 0-3 are allocated by `V_Init`; screen 4 is not allocated and was left for potential future use.

| Index | Use |
|-------|-----|
| 0 | The primary display buffer. Copied to the video hardware each frame by `I_UpdateScreen`. |
| 1 | Secondary buffer used for intermission background storage, double-buffering, and screen wipes. |
| 2 | Tertiary buffer, used for wipe source frames and temporary compositing. |
| 3 | Quaternary buffer, used similarly for wipe effects. |
| 4 | Not allocated. Pointer remains NULL after `V_Init`. |

---

### `dirtybox[4]`

```c
int dirtybox[4];
```

**Type:** Array of 4 ints, interpreted as a bounding box in the format `[top, bottom, left, right]` (using the `m_bbox.h` BOXTOP/BOXBOTTOM/BOXLEFT/BOXRIGHT convention).

**Purpose:** Tracks the rectangular region of screen 0 that has been modified since the last frame. Only modified regions need to be sent to the display hardware, which can save time on platforms that support partial screen updates. Functions that write to screen 0 call `V_MarkRect` to expand this box to include their modified area.

---

### `gammatable[5][256]`

```c
byte gammatable[5][256];
```

**Type:** 2D array of bytes, 5 rows of 256 entries each.

**Purpose:** Gamma correction lookup tables. Each row is a complete remapping table for one of the 5 gamma levels (0 through 4). To apply gamma correction, each palette index is passed through the current `gammatable[usegamma]` entry before display. Gamma level 0 is the identity (linear) mapping; higher levels progressively brighten the image.

**Implementation notes:**
- Gamma level 0: Maps input byte n to output byte n (linear, no correction). Slightly adjusted: value 0 maps to 1 (no true black).
- Gamma levels 1-4: Progressively compress the dynamic range of dark values, making the image appear brighter. Implemented as piecewise-linear approximations of gamma curves.
- The comment "Now where did these came from?" in the source reflects that the exact values were tuned empirically rather than derived from a mathematical formula.

The gamma tables work in conjunction with `usegamma` to allow the player to adjust display brightness. The actual palette sent to the hardware is computed by running the DOOM palette colors through `gammatable[usegamma]`.

---

### `usegamma`

```c
int usegamma;
```

**Type:** `int`.

**Purpose:** The currently selected gamma correction level. Valid range is 0 (no correction) through 4 (maximum brightness boost). Changed at runtime via the F11 key. Persists across sessions via the config file (`default.cfg`).

---

### `rcsid` (static)

```c
static const char rcsid[] = "$Id: v_video.c,v 1.5 1997/02/03 22:45:13 b1 Exp $";
```

**Type:** `static const char[]`.

**Purpose:** RCS (Revision Control System) version identification string. Embedded in the binary for version tracking. Has no runtime effect.

---

## Functions

### `V_MarkRect`

```c
void V_MarkRect(int x, int y, int width, int height);
```

**Purpose:** Expands the `dirtybox` bounding box to include the specified rectangular region. Called whenever any code draws to screen 0 so that the hardware update routine knows which pixels have changed.

**Parameters:**
- `x` - Left edge of the dirty rectangle (screen pixel coordinate).
- `y` - Top edge of the dirty rectangle (screen pixel coordinate).
- `width` - Width of the dirty rectangle in pixels.
- `height` - Height of the dirty rectangle in pixels.

**Return value:** None.

**Key logic:** Delegates to `M_AddToBox(dirtybox, x, y)` and `M_AddToBox(dirtybox, x+width-1, y+height-1)` from `m_bbox.h`. These calls expand the dirty box to encompass both corners of the new rectangle.

---

### `V_CopyRect`

```c
void V_CopyRect(int srcx, int srcy, int srcscrn,
                int width, int height,
                int destx, int desty, int destscrn);
```

**Purpose:** Copies a rectangular block of pixels from one screen buffer to another. Used for background preservation, double-buffering, and compositing operations.

**Parameters:**
- `srcx`, `srcy` - Top-left corner of the source rectangle in the source screen.
- `srcscrn` - Index (0-4) of the source screen buffer.
- `width`, `height` - Dimensions of the rectangle to copy.
- `destx`, `desty` - Top-left corner where the rectangle will be placed in the destination screen.
- `destscrn` - Index (0-4) of the destination screen buffer.

**Return value:** None.

**Key logic:** Computes source and destination byte pointers using `SCREENWIDTH` as the stride: `src = screens[srcscrn] + SCREENWIDTH * srcy + srcx`. Copies one row per iteration using `memcpy`. If writing to screen 0, `V_MarkRect` is called to update the dirty box.

Under `RANGECHECK` (debug build), validates all parameters and calls `I_Error` if they are out of range.

---

### `V_DrawPatch`

```c
void V_DrawPatch(int x, int y, int scrn, patch_t* patch);
```

**Purpose:** Draws a DOOM-format patch graphic to a screen buffer. This is the standard function for all 2D graphic rendering in DOOM — menus, status bar elements, HUD widgets, title screens, and intermission overlays all go through this function.

**Parameters:**
- `x` - Horizontal screen position for the patch's origin (before leftoffset adjustment).
- `y` - Vertical screen position for the patch's origin (before topoffset adjustment).
- `scrn` - Screen buffer index (0-4) to draw into.
- `patch` - Pointer to a `patch_t` structure in WAD-loaded memory.

**Return value:** None.

**Key logic:**

1. Adjusts `x` and `y` by the patch's `leftoffset` and `topoffset` fields: `y -= SHORT(patch->topoffset); x -= SHORT(patch->leftoffset);`. These offsets allow patch origins to be placed at logical reference points (e.g., the weapon sprite origin at the player's hands) rather than the top-left corner.

2. Under `RANGECHECK`, validates bounds. Notably, an out-of-bounds patch does **not** cause a fatal error (just a printed warning) to handle cases like TNT.WAD's oversized patches.

3. Iterates column by column. For each column, follows the linked list of `column_t` posts (each post has a `topdelta` specifying vertical gap, and `length` pixels of data). Draws each post's pixels vertically into the screen buffer.

4. Calls `V_MarkRect` to mark the dirty region only if drawing to screen 0.

**Patch column loop detail:**

```c
while (column->topdelta != 0xff) {
    source = (byte *)column + 3;        // pixel data starts 3 bytes into post header
    dest = desttop + column->topdelta * SCREENWIDTH;
    count = column->length;
    while (count--)
    {
        *dest = *source++;
        dest += SCREENWIDTH;            // advance vertically
    }
    column = (column_t*)((byte*)column + column->length + 4);
}
```

`topdelta == 0xFF` is the sentinel value marking the end of a column's post list.

---

### `V_DrawPatchFlipped`

```c
void V_DrawPatchFlipped(int x, int y, int scrn, patch_t* patch);
```

**Purpose:** Draws a patch horizontally mirrored. Used by the status bar to render the "mirror" face in deathmatch, and potentially for other symmetric graphics.

**Parameters:** Same as `V_DrawPatch`.

**Return value:** None.

**Key logic:** Identical to `V_DrawPatch` except that columns are iterated in reverse order: `patch->columnofs[w-1-col]` is used instead of `patch->columnofs[col]`. This reads the columns from right-to-left while still writing them left-to-right on screen, producing a horizontal mirror image. Not declared in `v_video.h` (internal use only).

---

### `V_DrawPatchDirect`

```c
void V_DrawPatchDirect(int x, int y, int scrn, patch_t* patch);
```

**Purpose:** Originally intended as a faster "direct to VGA hardware" patch drawing path for the DOS version, bypassing the software framebuffer. In the Linux port this is a trivial wrapper around `V_DrawPatch`.

**Parameters:** Same as `V_DrawPatch`.

**Return value:** None.

**Key logic:** Simply calls `V_DrawPatch(x, y, scrn, patch)`. The original EGA/VGA plane-switching code is preserved in a `/* ... */` comment block for historical reference, showing the DOS-specific implementation that wrote directly to VGA planes using the sequence controller port (`SC_INDEX`).

---

### `V_DrawBlock`

```c
void V_DrawBlock(int x, int y, int scrn, int width, int height, byte* src);
```

**Purpose:** Copies a contiguous rectangular block of pixels from an arbitrary memory buffer into a screen buffer. Unlike `V_DrawPatch`, this operates on raw linear pixel data with no column/post structure and no transparency.

**Parameters:**
- `x`, `y` - Top-left destination coordinates in the screen buffer.
- `scrn` - Destination screen buffer index.
- `width`, `height` - Dimensions of the block.
- `src` - Pointer to the source pixel data. The source is assumed to be a packed, row-major buffer with `width` bytes per row (no padding).

**Return value:** None.

**Key logic:** Computes `dest = screens[scrn] + y*SCREENWIDTH + x` and copies `height` rows of `width` bytes each using `memcpy`. Source pointer advances by `width` each row; destination pointer advances by `SCREENWIDTH` each row (since the screen buffer row stride is always `SCREENWIDTH`, which may be larger than `width`). Calls `V_MarkRect` for the dirty region.

---

### `V_GetBlock`

```c
void V_GetBlock(int x, int y, int scrn, int width, int height, byte* dest);
```

**Purpose:** The inverse of `V_DrawBlock`. Reads a rectangular region from a screen buffer into an arbitrary destination memory buffer. Used to save backgrounds before drawing overlays that need to be erased later.

**Parameters:**
- `x`, `y` - Top-left coordinates of the region to read from the screen.
- `scrn` - Source screen buffer index.
- `width`, `height` - Dimensions of the region to read.
- `dest` - Destination buffer to receive the pixels. Must be at least `width * height` bytes.

**Return value:** None.

**Key logic:** Symmetric to `V_DrawBlock`: reads `height` rows of `width` bytes, with source advancing by `SCREENWIDTH` and destination advancing by `width`. Does not call `V_MarkRect` (reading is non-destructive).

---

### `V_Init`

```c
void V_Init(void);
```

**Purpose:** Allocates and initializes the five software screen buffers. Called once at engine startup before any rendering occurs.

**Parameters:** None.

**Return value:** None.

**Key logic:** Calls `I_AllocLow(SCREENWIDTH * SCREENHEIGHT * 4)` to allocate a single contiguous block of memory large enough for 4 screens (the comment "stick these in low DOS memory on PCs" is a historical artifact). Divides the block into 4 equal regions and assigns pointers:

```c
for (i = 0; i < 4; i++)
    screens[i] = base + i * SCREENWIDTH * SCREENHEIGHT;
```

Screen 4 (`screens[4]`) is not allocated and remains uninitialized (effectively NULL or garbage).

---

## Data Structures

This file does not define any new data structures. It uses:

- `patch_t` - Defined in `r_data.h`. The patch image format with header fields `leftoffset`, `topoffset`, `width`, `height`, and `columnofs[]`.
- `column_t` - Defined in `r_data.h`. Represents a single vertical "post" in a patch column, with fields `topdelta` and `length`.

---

## Dependencies

| File | Reason |
|------|--------|
| `i_system.h` | `I_AllocLow` (allocates the screen buffers), `I_Error` (fatal error reporting). |
| `r_local.h` | Transitively provides `patch_t`, `column_t`, and rendering constants. |
| `doomdef.h` | `SCREENWIDTH`, `SCREENHEIGHT`, and `RANGECHECK` flag. |
| `doomdata.h` | Basic DOOM data type declarations. |
| `m_bbox.h` | `M_AddToBox` used by `V_MarkRect` to expand the dirty bounding box. |
| `m_swap.h` | `SHORT()` and `LONG()` byte-swap macros used when reading patch header fields (WAD data is little-endian). |
| `v_video.h` | Self-header, providing the public interface declarations. |
