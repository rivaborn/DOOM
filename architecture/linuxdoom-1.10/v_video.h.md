# File Overview

`v_video.h` is the public interface header for DOOM's 2D video rendering layer. It declares the screen buffer array, the dirty box, the gamma correction tables, and all the functions used to draw patches, copy rectangles, and block-blit pixels to the software framebuffers. Any module that needs to draw 2D graphics to the screen includes this header.

This header is included by the status bar (`st_stuff.c`), the intermission screen (`wi_stuff.c`), the menu (`m_menu.c`), the finale (`f_finale.c`), the automap (`am_map.c`), the HUD (`hu_stuff.c`), and the wipe system (`f_wipe.c`).

---

## Global Variables

### `screens[5]`

```c
extern byte* screens[5];
```

**Type:** Array of 5 byte pointers (one per software framebuffer).

**Purpose:** The five software framebuffers. Screen 0 is the primary display buffer; screens 1-3 are auxiliary buffers. Each allocated screen is `SCREENWIDTH * SCREENHEIGHT` bytes (64,000 bytes at 320x200). See `v_video.c.md` for per-screen usage details.

---

### `dirtybox[4]`

```c
extern int dirtybox[4];
```

**Type:** Array of 4 ints representing a bounding box (top, bottom, left, right per `m_bbox.h` conventions).

**Purpose:** Tracks the rectangular dirty region of screen 0 that needs to be sent to the display hardware. Expanded by `V_MarkRect` whenever screen 0 is modified.

---

### `gammatable[5][256]`

```c
extern byte gammatable[5][256];
```

**Type:** 2D byte array, 5 gamma levels each with 256 remapping entries.

**Purpose:** Gamma correction lookup tables. `gammatable[usegamma][n]` gives the corrected output value for palette index `n` at the current gamma level. Level 0 is approximately linear; level 4 is the brightest. Used by the palette/display code to adjust brightness without changing the underlying framebuffer.

---

### `usegamma`

```c
extern int usegamma;
```

**Type:** `int`.

**Purpose:** Currently active gamma correction level (0-4). Toggled by F11 at runtime. Stored in `default.cfg` for persistence.

---

## Functions

### `V_Init`

```c
void V_Init(void);
```

**Purpose:** Allocates all software screen buffers and initializes the `screens[]` array. Must be called before any drawing functions are used.

**Parameters:** None.

**Return value:** None.

---

### `V_CopyRect`

```c
void V_CopyRect(int srcx, int srcy, int srcscrn,
                int width, int height,
                int destx, int desty, int destscrn);
```

**Purpose:** Copies a rectangular region from one screen buffer to another.

**Parameters:**
- `srcx`, `srcy` - Top-left corner of the source rectangle.
- `srcscrn` - Source screen buffer index (0-4).
- `width`, `height` - Dimensions of the region.
- `destx`, `desty` - Top-left corner of the destination rectangle.
- `destscrn` - Destination screen buffer index (0-4).

**Return value:** None.

---

### `V_DrawPatch`

```c
void V_DrawPatch(int x, int y, int scrn, patch_t* patch);
```

**Purpose:** Renders a WAD patch graphic (DOOM's native column-based masked sprite format) to a screen buffer. This is the primary 2D drawing function for all non-3D content.

**Parameters:**
- `x`, `y` - Screen position for the patch origin point (adjusted by the patch's own `leftoffset`/`topoffset` internally).
- `scrn` - Screen buffer index (0-4).
- `patch` - Pointer to a `patch_t` structure loaded from the WAD.

**Return value:** None.

**Note:** Under `RANGECHECK`, an out-of-range patch only prints a warning (not a fatal error), to handle real-world WAD files with slightly oversized graphics.

---

### `V_DrawPatchDirect`

```c
void V_DrawPatchDirect(int x, int y, int scrn, patch_t* patch);
```

**Purpose:** Originally a faster direct-to-hardware patch drawing path for the DOS VGA implementation. In the Linux port, this is identical to `V_DrawPatch`.

**Parameters:** Same as `V_DrawPatch`.

**Return value:** None.

---

### `V_DrawBlock`

```c
void V_DrawBlock(int x, int y, int scrn, int width, int height, byte* src);
```

**Purpose:** Copies a rectangular block of raw linear pixel data (no column/post structure, no transparency) into a screen buffer.

**Parameters:**
- `x`, `y` - Destination top-left corner.
- `scrn` - Destination screen buffer index.
- `width`, `height` - Dimensions of the block.
- `src` - Source pixel buffer, packed row-major with `width` bytes per row.

**Return value:** None.

---

### `V_GetBlock`

```c
void V_GetBlock(int x, int y, int scrn, int width, int height, byte* dest);
```

**Purpose:** Reads a rectangular region from a screen buffer into a flat pixel buffer. The inverse of `V_DrawBlock`.

**Parameters:**
- `x`, `y` - Top-left corner of the source region in the screen.
- `scrn` - Source screen buffer index.
- `width`, `height` - Dimensions of the region to read.
- `dest` - Destination buffer to receive the pixels.

**Return value:** None.

---

### `V_MarkRect`

```c
void V_MarkRect(int x, int y, int width, int height);
```

**Purpose:** Expands `dirtybox` to include the specified rectangle. Should be called whenever a function writes to screen 0. Does not need to be called for writes to screens 1-4 since those are not directly displayed.

**Parameters:**
- `x`, `y` - Top-left corner of the modified region.
- `width`, `height` - Dimensions of the modified region.

**Return value:** None.

---

## Data Structures

This header uses `patch_t` (from `r_data.h`) and `byte` (from `doomtype.h`). It does not define any new types.

---

## Macros

| Macro | Value | Purpose |
|-------|-------|---------|
| `CENTERY` | `SCREENHEIGHT / 2` | Vertical center of the screen (100 at standard 200-pixel height). Used by various drawing routines as a reference point. |

---

## Dependencies

| File | Reason |
|------|--------|
| `doomtype.h` | Provides `byte` and `boolean` types. |
| `doomdef.h` | Provides `SCREENHEIGHT` used in the `CENTERY` macro. |
| `r_data.h` | Provides `patch_t` (the WAD patch format structure) used in the patch drawing function signatures. |
