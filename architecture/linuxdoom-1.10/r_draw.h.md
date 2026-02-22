# File Overview

**File:** `linuxdoom-1.10/r_draw.h`
**Module:** Renderer - Low-Level Pixel Drawing (Public Interface)

This header is the public interface for `r_draw.c`, DOOM's lowest-level rendering layer. It declares the column and span rendering functions, the shared state variables they read (set up by higher-level renderer code before each call), buffer initialization routines, and color translation infrastructure.

DOOM's renderer does not write individual pixels directly. Instead, all visible geometry is ultimately decomposed into either vertical **columns** (for walls and sprites) or horizontal **spans** (for floors and ceilings). The actual pixel-writing routines are selected at runtime through function pointers in `r_main.h` (`colfunc`, `spanfunc`, etc.), allowing transparent swap-in of alternative implementations such as the fuzz/invisibility effect or low-detail mode variants.

The column and span functions operate by reading from a shared set of global input variables declared here. The higher-level code (e.g., `r_segs.c`, `r_plane.c`, `r_things.c`) fills these globals and then calls the appropriate function pointer.

---

## Global Variables

### Column renderer inputs

These variables must be set by the caller before invoking any column-drawing function.

| Type             | Name           | Purpose |
|------------------|----------------|---------|
| `lighttable_t*`  | `dc_colormap`  | Pointer into the colormap/lighting table used to translate texture palette indices to the final display palette. A `NULL` colormap signals the shadow/fuzz effect (not read directly by the column function; that is handled by switching `colfunc` to `fuzzcolfunc`). |
| `int`            | `dc_x`         | Screen X column to draw (0-based). |
| `int`            | `dc_yl`        | Top screen Y row of the column segment to draw (inclusive). |
| `int`            | `dc_yh`        | Bottom screen Y row of the column segment to draw (inclusive). |
| `fixed_t`        | `dc_iscale`    | Texture coordinate step per screen row (inverse scale). Computed as `FRACUNIT / column_height_in_pixels`. Larger values sample the texture more coarsely (far-away walls). |
| `fixed_t`        | `dc_texturemid`| Fixed-point texture row coordinate corresponding to the center of the view (`centery`). The renderer walks upward and downward from this point using `dc_iscale` to determine which texture row to read for each screen row. |
| `byte*`          | `dc_source`    | Pointer to the first byte of the texture column data (a 128-byte or 256-byte tall column). Indexed with the current texture fraction to retrieve a palette index. |

### Span renderer inputs

These variables must be set by the caller before invoking any span-drawing function.

| Type             | Name          | Purpose |
|------------------|---------------|---------|
| `int`            | `ds_y`        | Screen Y row of the horizontal span to draw. |
| `int`            | `ds_x1`       | Leftmost X column of the span (inclusive). |
| `int`            | `ds_x2`       | Rightmost X column of the span (inclusive). |
| `lighttable_t*`  | `ds_colormap` | Lighting/colormap table for the span. Same semantics as `dc_colormap`. |
| `fixed_t`        | `ds_xfrac`    | Starting horizontal texture coordinate (in fixed-point). Advances by `ds_xstep` per column. |
| `fixed_t`        | `ds_yfrac`    | Starting vertical texture coordinate (in fixed-point). Advances by `ds_ystep` per column. |
| `fixed_t`        | `ds_xstep`    | Per-column increment for `ds_xfrac`. Derived from the perspective of the floor at height `ds_y`. |
| `fixed_t`        | `ds_ystep`    | Per-column increment for `ds_yfrac`. |
| `byte*`          | `ds_source`   | Pointer to the start of a 64x64 flat tile image. Indexed as `ds_source[(yfrac >> 26) * 64 + (xfrac >> 26)]` (upper 6 bits of the fraction give the 0-63 tile coordinate). |

### Color translation

| Type    | Name               | Purpose |
|---------|--------------------|---------|
| `byte*` | `translationtables`| Base of the color translation table block. Contains 3 x 256-byte tables (offset by multiples of 256) for the green, indigo, brown, and red player shirt colors. Allocated by `R_InitTranslationTables`. |
| `byte*` | `dc_translation`   | Pointer to the specific 256-byte translation table to use for the current column. Set by `R_DrawVisSprite` to `translationtables - 256 + (colorclass * 256)` before calling `R_DrawTranslatedColumn`. |

---

## Functions

### Column Renderers

#### `R_DrawColumn`

```c
void R_DrawColumn(void);
```

Standard wall/sprite column renderer. Reads `dc_colormap`, `dc_x`, `dc_yl`, `dc_yh`, `dc_iscale`, `dc_texturemid`, and `dc_source`. Writes one vertical strip of pixels into the screen buffer. This is the hot path for normal solid wall and sprite rendering.

#### `R_DrawColumnLow`

```c
void R_DrawColumnLow(void);
```

Low-detail mode variant of `R_DrawColumn`. Writes doubled-width pixels (two adjacent columns at once) to simulate the 160x200 blocky detail mode selectable from the options menu (`detailshift == 1`). Selected via the `colfunc` pointer by `R_SetViewSize`.

#### `R_DrawFuzzColumn`

```c
void R_DrawFuzzColumn(void);
```

Implements the Spectre/partial-invisibility effect (the "fuzz" effect). Instead of reading from `dc_source` normally, it applies a fixed jitter pattern (`fuzzoffset`) to the screen buffer address, sampling nearby already-rendered pixels and darkening them through `colormaps`. This creates the characteristic noisy dark distortion of invisible entities. Selected when `vis->colormap == NULL` in `R_DrawVisSprite`.

#### `R_DrawFuzzColumnLow`

```c
void R_DrawFuzzColumnLow(void);
```

Low-detail mode variant of `R_DrawFuzzColumn`.

#### `R_DrawTranslatedColumn`

```c
void R_DrawTranslatedColumn(void);
```

Column renderer with color remapping. Before applying `dc_colormap`, passes each source pixel through the `dc_translation` table to remap palette colors (used to render player sprites in non-green team colors: red, blue, indigo, brown shirts). Selected when the `MF_TRANSLATION` flag is set on the sprite's `mobj_t`.

#### `R_DrawTranslatedColumnLow`

```c
void R_DrawTranslatedColumnLow(void);
```

Low-detail mode variant of `R_DrawTranslatedColumn`.

---

### Span Renderers

#### `R_DrawSpan`

```c
void R_DrawSpan(void);
```

Renders one horizontal span of floor or ceiling pixels. Uses `ds_xfrac`/`ds_yfrac` with `ds_xstep`/`ds_ystep` to walk across a 64x64 flat texture with affine (non-perspective-corrected) mapping. No fuzz effect is defined for spans because invisible creatures' shadows are column-based and do not require floor rendering.

#### `R_DrawSpanLow`

```c
void R_DrawSpanLow(void);
```

Low-detail mode variant of `R_DrawSpan`. Skips every other column (or doubles writes) to produce the 160x200 blocky floor appearance.

---

### Buffer and Initialization Functions

#### `R_VideoErase`

```c
void R_VideoErase(unsigned ofs, int count);
```

Erases `count` bytes of the video buffer starting at byte offset `ofs`. Used by `R_FillBackScreen` and `R_DrawViewBorder` to clear the border region around the viewport when the view is smaller than full screen.

**Parameters:**
- `ofs` - Byte offset from the start of the screen buffer.
- `count` - Number of bytes to erase (fill with the background flat pattern).

#### `R_InitBuffer`

```c
void R_InitBuffer(int width, int height);
```

Initializes the rendering buffer parameters when the view size changes. Sets `viewwindowx`, `viewwindowy`, and the `ylookup[]` / `columnofs[]` address lookup tables that all column and span functions use to compute their destination pointer in the screen buffer.

**Parameters:**
- `width` - New viewport width in pixels.
- `height` - New viewport height in pixels.

#### `R_InitTranslationTables`

```c
void R_InitTranslationTables(void);
```

Allocates and populates the `translationtables` block. Creates 3 translation tables that remap the green player palette range to the alternate shirt colors. Called once at startup from `R_Init`.

#### `R_FillBackScreen`

```c
void R_FillBackScreen(void);
```

Tiles the background flat (`FLOOR7_2` by default) behind the viewport when the view is smaller than the full screen. Called from `D_Display` each frame when `scaledviewwidth < SCREENWIDTH`.

#### `R_DrawViewBorder`

```c
void R_DrawViewBorder(void);
```

Draws the decorative border patches (corner and edge graphics from the `INTERPIC`-style set) around the viewport frame. Called after `R_FillBackScreen` when the view is windowed. Uses `R_VideoErase` to do the actual pixel copying.

---

## Dependencies

| File | Reason |
|------|--------|
| `doomdef.h` (via `r_local.h`) | `byte`, `boolean`, `fixed_t`, `SCREENWIDTH` |
| `r_defs.h` (via `r_local.h`) | `lighttable_t` type definition |

Include guard: `__R_DRAW__`

**Note on function pointers:** The column and span functions declared here are typically not called directly. Instead they are called through the function pointers `colfunc`, `basecolfunc`, `fuzzcolfunc`, and `spanfunc` declared in `r_main.h`. These pointers are set during `R_SetViewSize` / `R_ExecuteSetViewSize` to select the appropriate normal or low-detail variant.
