# File Overview

`r_draw.c` contains the innermost pixel-writing routines of DOOM's software renderer — the **column drawers** and **span drawers**. All other rendering modules (BSP traversal, wall rendering, plane rendering, sprite rendering) calculate *what* to draw and *where*; this file does the actual work of writing pixels into the framebuffer.

There are two fundamental primitives:

- **Column drawing**: Writing a vertical strip of pixels (for walls and sprites). Because DOOM uses a cylindrical projection with a vertical eye axis, all wall columns and sprite columns have constant depth along their height and can be rendered with a simple fixed-point step loop.
- **Span drawing**: Writing a horizontal strip of pixels (for floors and ceilings). Floor/ceiling rows also have constant depth across their width at a given screen y-coordinate, allowing a linear walk through the texture using fixed-point step increments.

The file also provides the **framebuffer address lookup tables** (`ylookup[]`, `columnofs[]`), **view window management** (`R_InitBuffer`, `R_FillBackScreen`, `R_DrawViewBorder`), and the **color translation** system for multiplayer suit colors.

## Global Variables

### View geometry

| Type | Name | Description |
|---|---|---|
| `byte*` | `viewimage` | Pointer to the active view buffer (set during `R_InitBuffer`). |
| `int` | `viewwidth` | Width of the rendered view in pixels (may be less than SCREENWIDTH in windowed mode). |
| `int` | `scaledviewwidth` | Width at full (non-low-detail) resolution; `viewwidth << detailshift`. |
| `int` | `viewheight` | Height of the rendered view in pixels. |
| `int` | `viewwindowx` | Left edge of the view window within the framebuffer. |
| `int` | `viewwindowy` | Top edge of the view window within the framebuffer. |

### Framebuffer address lookup tables

| Type | Name | Description |
|---|---|---|
| `byte* ` | `ylookup[MAXHEIGHT]` | `ylookup[y]` = pointer to the start of screen row `y` in `screens[0]`. Avoids a multiply by `SCREENWIDTH` in the inner loops. |
| `int` | `columnofs[MAXWIDTH]` | `columnofs[x]` = column offset from `ylookup[y]` to reach pixel `(x, y)`. Accounts for `viewwindowx` centering. |

### Color translation

| Type | Name | Description |
|---|---|---|
| `byte` | `translations[3][256]` | Three 256-entry color translation tables (gray, brown, red) that remap the player sprite's green color ramp for multiplayer suits. |

### Column drawer parameters (shared with `r_segs.c`, `r_things.c` via `r_draw.h`)

| Type | Name | Description |
|---|---|---|
| `lighttable_t*` | `dc_colormap` | Current colormap (256-entry palette remap) to apply to column pixels. |
| `int` | `dc_x` | Screen X column being drawn. |
| `int` | `dc_yl` | Top screen Y of the column span. |
| `int` | `dc_yh` | Bottom screen Y of the column span. |
| `fixed_t` | `dc_iscale` | Inverse of the rendering scale: fixed-point step through texture per output pixel. `dc_iscale = 0xffffffff / scale`. |
| `fixed_t` | `dc_texturemid` | Texture row at the center of the screen for this column. |
| `byte*` | `dc_source` | Pointer to the texture column pixel data. |
| `int` | `dccount` | Profiling counter (unused in release builds). |

### Fuzz effect

| Type | Name | Description |
|---|---|---|
| `int` | `fuzzoffset[FUZZTABLE]` | Static table of 50 pixel offsets (`+SCREENWIDTH` or `-SCREENWIDTH`) defining the spectre/invisibility dithering pattern. |
| `int` | `fuzzpos` | Current position in the `fuzzoffset` table; wraps at `FUZZTABLE` (50). |

### Color translation (column drawer)

| Type | Name | Description |
|---|---|---|
| `byte*` | `dc_translation` | Points to the active 256-entry translation table for the current sprite being drawn with `R_DrawTranslatedColumn`. |
| `byte*` | `translationtables` | Base pointer for the three 256-entry translation tables allocated in `R_InitTranslationTables`. 256-byte aligned. |

### Span drawer parameters (shared via `r_draw.h`)

| Type | Name | Description |
|---|---|---|
| `int` | `ds_y` | Screen Y row for the current span. |
| `int` | `ds_x1` | Left screen X of the span. |
| `int` | `ds_x2` | Right screen X of the span. |
| `lighttable_t*` | `ds_colormap` | Colormap for flat lighting. |
| `fixed_t` | `ds_xfrac` | Current U texture coordinate (fixed-point), updated per pixel. |
| `fixed_t` | `ds_yfrac` | Current V texture coordinate (fixed-point), updated per pixel. |
| `fixed_t` | `ds_xstep` | U step per horizontal pixel. |
| `fixed_t` | `ds_ystep` | V step per horizontal pixel. |
| `byte*` | `ds_source` | Pointer to the 64x64 flat texture data. |
| `int` | `dscount` | Profiling counter (unused in release builds). |

## Functions

### `R_DrawColumn`

```c
void R_DrawColumn(void)
```

**Purpose:** The primary vertical column drawer. Draws a textured, lit wall or sprite column at screen column `dc_x` from row `dc_yl` to `dc_yh`. Applies colormap lighting.

**Parameters:** None (all inputs via global `dc_*` variables).

**Return Value:** `void`

**Key Logic:**

1. `count = dc_yh - dc_yl`. If negative, return.
2. `dest = ylookup[dc_yl] + columnofs[dc_x]` — framebuffer address.
3. `fracstep = dc_iscale` — step through the texture per screen pixel.
4. `frac = dc_texturemid + (dc_yl - centery) * fracstep` — starting texture row.
5. Inner loop: `*dest = dc_colormap[dc_source[(frac >> FRACBITS) & 127]]`
   - `(frac >> FRACBITS) & 127` converts the fixed-point fraction to a texture row index (0–127, wrapping for 128-pixel textures).
   - The colormap lookup implements distance-based dimming: `dc_colormap[pixel]` remaps the raw palette index to a darker version.
   - `dest += SCREENWIDTH` advances to the next row.
   - `frac += fracstep` advances through the texture.

The `& 127` mask implies all DOOM wall textures are at most 128 pixels tall. The comment notes this is a "DDA-like scaling" — it is essentially a fixed-point bresenham walk through the texture.

---

### `R_DrawColumnLow`

```c
void R_DrawColumnLow(void)
```

**Purpose:** Low-detail (blocky) version of `R_DrawColumn`. Doubles the pixel width by writing to two adjacent columns (`dc_x*2` and `dc_x*2 + 1`).

**Parameters:** None.

**Return Value:** `void`

**Key Logic:** Same as `R_DrawColumn` but: `dc_x <<= 1`, then writes both `dest` and `dest2` (the adjacent column) with the same pixel value each row.

---

### `R_DrawFuzzColumn`

```c
void R_DrawFuzzColumn(void)
```

**Purpose:** Draws the spectre/partial-invisibility effect. Instead of reading from a texture, it reads an already-rendered adjacent pixel, darkens it through colormap index 6, and writes it back. The `fuzzoffset` table alternates the source pixel between one row above and below the destination.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:**

1. Clamps `dc_yl` to 1 and `dc_yh` to `viewheight - 2` to prevent reading outside the framebuffer.
2. `dest = ylookup[dc_yl] + columnofs[dc_x]`.
3. Inner loop: `*dest = colormaps[6*256 + dest[fuzzoffset[fuzzpos]]]`
   - `dest[fuzzoffset[fuzzpos]]` reads a pixel offset by `±SCREENWIDTH` (one row above or below) in the framebuffer.
   - `colormaps[6*256 + ...]` applies colormap 6 (~75% brightness) to darken it.
   - `fuzzpos` cycles through the 50-entry `fuzzoffset` table.

The result is a shimmering, darkened distortion of whatever was already rendered behind the spectre — DOOM's iconic "partial invisibility" effect.

---

### `R_DrawTranslatedColumn`

```c
void R_DrawTranslatedColumn(void)
```

**Purpose:** Like `R_DrawColumn` but applies an additional color-translation step before the colormap lookup. Used to draw player sprites in multiplayer with non-green suit colors.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:** Inner loop: `*dest = dc_colormap[dc_translation[dc_source[frac >> FRACBITS]]]`
- `dc_source[frac >> FRACBITS]` fetches the raw sprite pixel.
- `dc_translation[pixel]` remaps the color (green → gray/brown/red for the other players).
- `dc_colormap[...]` applies sector lighting.

---

### `R_InitTranslationTables`

```c
void R_InitTranslationTables(void)
```

**Purpose:** Allocates and builds the three 256-entry player color translation tables (green-to-gray, green-to-brown, green-to-red).

**Parameters:** None.

**Return Value:** `void`

**Key Logic:**

1. Allocates `256*3 + 255` bytes and 256-byte-aligns the pointer.
2. For palette indices `0x70–0x7F` (the green ramp in DOOM's PLAYPAL): maps to `0x60+i&0xF` (gray), `0x40+i&0xF` (brown), `0x20+i&0xF` (red) in the three tables respectively.
3. All other indices map to themselves (identity).

---

### `R_DrawSpan`

```c
void R_DrawSpan(void)
```

**Purpose:** The primary horizontal span drawer. Draws a textured, lit floor or ceiling span across screen row `ds_y` from column `ds_x1` to `ds_x2`.

**Parameters:** None (all inputs via `ds_*` globals).

**Return Value:** `void`

**Key Logic:**

1. `dest = ylookup[ds_y] + columnofs[ds_x1]`.
2. Inner loop per pixel:
   - `spot = ((yfrac >> (16-6)) & (63*64)) + ((xfrac >> 16) & 63)`
   - This computes a 2D index into the 64x64 flat texture: the upper 6 bits of `yfrac` (after shifting) give the V row (multiplied by 64), and the upper 6 bits of `xfrac` give the U column.
   - `*dest++ = ds_colormap[ds_source[spot]]`
   - `xfrac += ds_xstep; yfrac += ds_ystep` — the fixed-point coordinates walk diagonally through the flat texture as the span progresses horizontally.

The step values are computed in `r_plane.c` based on the floor height and viewing angle, providing affine texture mapping for flat surfaces. This mapping is not perspective-correct but is exact for flat (level) surfaces viewed from a horizontal eye position.

---

### `R_DrawSpanLow`

```c
void R_DrawSpanLow(void)
```

**Purpose:** Low-detail version of `R_DrawSpan`. Doubles the horizontal pixel width by writing each computed texel twice.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:** Doubles `ds_x1` and `ds_x2`. Inner loop writes `*dest++` and `*dest++` with the same texel value before advancing `xfrac`/`yfrac`.

---

### `R_InitBuffer`

```c
void R_InitBuffer(int width, int height)
```

**Purpose:** Sets up the `ylookup[]` and `columnofs[]` address lookup tables for the given view window size. Called whenever the view size changes.

**Parameters:**
- `width` (`int`): View width in pixels.
- `height` (`int`): View height in pixels.

**Return Value:** `void`

**Key Logic:**

1. `viewwindowx = (SCREENWIDTH - width) >> 1` — centers the view horizontally.
2. `columnofs[i] = viewwindowx + i` for each column.
3. `viewwindowy`: 0 if full-width, else centered vertically above the status bar.
4. `ylookup[i] = screens[0] + (i + viewwindowy) * SCREENWIDTH` for each row.

---

### `R_FillBackScreen`

```c
void R_FillBackScreen(void)
```

**Purpose:** Fills the background screen (`screens[1]`) with a tiling floor texture pattern and draws the beveled border around the view window. Used when the view window is smaller than the full screen.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:** Tiles `FLOOR7_2` (DOOM) or `GRNROCK` (DOOM II) across the entire background except the view window. Then draws `brdr_t`, `brdr_b`, `brdr_l`, `brdr_r` border patches and the four corner patches (`brdr_tl`, `brdr_tr`, `brdr_bl`, `brdr_br`) using `V_DrawPatch`.

---

### `R_VideoErase`

```c
void R_VideoErase(unsigned ofs, int count)
```

**Purpose:** Copies `count` bytes from `screens[1]` (background) to `screens[0]` (view) starting at byte offset `ofs`. Used to restore border areas after resizing.

**Parameters:**
- `ofs` (`unsigned`): Byte offset into both screen buffers.
- `count` (`int`): Number of bytes to copy.

**Return Value:** `void`

---

### `R_DrawViewBorder`

```c
void R_DrawViewBorder(void)
```

**Purpose:** Copies the border regions from `screens[1]` to `screens[0]` to redraw the area around a sub-fullscreen view. Called each frame when `scaledviewwidth < SCREENWIDTH`.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:** Erases top strip, bottom strip, and the left/right side strips column-by-column using `R_VideoErase`. Then marks the entire screen dirty via `V_MarkRect`.

## Data Structures

No new data structures are defined in this file. The locally defined constants are:

| Constant | Value | Purpose |
|---|---|---|
| `MAXWIDTH` | 1120 | Maximum supported screen width (for `ylookup`/`columnofs` array sizing) |
| `MAXHEIGHT` | 832 | Maximum supported screen height |
| `SBARHEIGHT` | 32 | Status bar height in pixels, used to center the view window above it |
| `FUZZTABLE` | 50 | Number of entries in the fuzz offset table |
| `FUZZOFF` | `SCREENWIDTH` | Pixel offset magnitude for fuzz effect (one row up or down) |

## Dependencies

| File | Reason |
|---|---|
| `doomdef.h` | `SCREENWIDTH`, `SCREENHEIGHT`, `byte`, `boolean`, `fixed_t` |
| `i_system.h` | `I_Error()` for range-check failures |
| `z_zone.h` | `Z_Malloc` for translation table allocation |
| `w_wad.h` | `W_CacheLumpName` for border patch loading |
| `r_local.h` | `centery`, `FRACBITS`, `FRACUNIT`, `lighttable_t`, and all other renderer definitions |
| `v_video.h` | `screens[]` framebuffer array, `V_DrawPatch`, `V_MarkRect` |
| `doomstat.h` | `gamemode` (commercial vs. shareware, for background texture selection), `colormaps` |
