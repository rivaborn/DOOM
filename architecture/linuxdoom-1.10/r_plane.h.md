# File Overview

**Path:** `linuxdoom-1.10/r_plane.h`
**Module:** Renderer - Floor and Ceiling Plane Public Interface Header

`r_plane.h` is the public interface header for `r_plane.c`. It exposes the declarations needed by other renderer modules - primarily `r_segs.c` and `r_bsp.c` - to interact with the visplane and floor/ceiling clipping subsystems.

The header declares:
- The `planefunction_t` function pointer type.
- The per-column floor and ceiling clip arrays that `r_segs.c` reads and writes.
- The `yslope[]` and `distscale[]` precomputed geometry tables.
- The full set of visplane management and rendering function prototypes.

This header is included by `r_local.h`, making its declarations available to all renderer source files.

---

## Global Variables

All variables below are declared `extern` here and defined in `r_plane.c`.

| Type | Name | Purpose |
|------|------|---------|
| `short*` | `lastopening` | Next free position in the `openings[]` flat buffer. Advances as sprite clip arrays and masked texture column data are allocated during BSP traversal. |
| `planefunction_t` | `floorfunc` | Unused function pointer; declared for a floor-drawing dispatch mechanism that was not implemented in the Linux release. |
| `planefunction_t` | `ceilingfunc_t` | Unused function pointer; same situation as `floorfunc`. Note the inconsistent `_t` suffix versus the `ceilingfunc` name used in `r_plane.c`. |
| `short` | `floorclip[SCREENWIDTH]` | Per-column floor clip boundary. Initialized to `viewheight` (bottom of screen) each frame by `R_ClearPlanes`. Decremented by `R_RenderSegLoop` as solid wall segments are rendered, so floors are never drawn above a wall's bottom edge. |
| `short` | `ceilingclip[SCREENWIDTH]` | Per-column ceiling clip boundary. Initialized to `-1` (above screen top) each frame by `R_ClearPlanes`. Incremented by `R_RenderSegLoop` as solid wall segments are rendered, so ceilings are never drawn below a wall's top edge. |
| `fixed_t` | `yslope[SCREENHEIGHT]` | Precomputed per-row distance scale: `(viewwidth/2 * FRACUNIT) / |row - centery|`. Approaches infinity at the horizon row and decreases toward the screen edges. Computed in `R_ExecuteSetViewSize` and constant for the life of a view size. |
| `fixed_t` | `distscale[SCREENWIDTH]` | Precomputed per-column perspective correction factor: `1 / cos(xtoviewangle[x])`. Corrects for the oblique angle when computing the world-space distance to a floor/ceiling point at column `x`. Computed in `R_ExecuteSetViewSize`. |

---

## Functions

### `R_InitPlanes`
```c
void R_InitPlanes(void)
```
**Purpose:** Startup stub. Called once from `R_Init`. The current implementation has an empty body; all runtime initialization is done by `R_ClearPlanes` at the start of each frame.

---

### `R_ClearPlanes`
```c
void R_ClearPlanes(void)
```
**Purpose:** Per-frame reset of all plane-system state. Called by `R_RenderPlayerView` before BSP traversal begins. Resets `floorclip[]`, `ceilingclip[]`, `lastvisplane`, `lastopening`, the distance cache, and the `basexscale`/`baseyscale` texture step bases.

---

### `R_MapPlane`
```c
void R_MapPlane(int y, int x1, int x2)
```
**Purpose:** Rasterizes a single horizontal floor or ceiling span from column `x1` to `x2` at screen row `y`. Sets all `ds_*` span-drawing globals (texture coordinates, step values, colormap) and then calls `spanfunc()`.

**Parameters:**
- `y` - Screen row.
- `x1` - Start column (inclusive).
- `x2` - End column (inclusive).

---

### `R_MakeSpans`
```c
void R_MakeSpans(int x, int t1, int b1, int t2, int b2)
```
**Purpose:** State machine for converting per-column visplane data into horizontal spans. Called by `R_DrawPlanes` as it scans left-to-right across each visplane. Closes spans that end at column `x-1` (calls `R_MapPlane`) and opens spans that begin at column `x` (records `spanstart[row] = x`).

**Parameters:**
- `x` - The current column being processed (the new column).
- `t1`, `b1` - Top and bottom of the occupied region in the previous column.
- `t2`, `b2` - Top and bottom of the occupied region in the current column.

---

### `R_DrawPlanes`
```c
void R_DrawPlanes(void)
```
**Purpose:** End-of-frame plane rasterization pass. Iterates all accumulated visplanes and renders each one. Sky planes are drawn as vertical columns using `colfunc()`; regular floor/ceiling planes are drawn as horizontal spans using `spanfunc()` via `R_MakeSpans` and `R_MapPlane`.

---

### `R_FindPlane`
```c
visplane_t* R_FindPlane(fixed_t height, int picnum, int lightlevel)
```
**Purpose:** Returns a visplane for the given `(height, picnum, lightlevel)` triple. If a matching visplane already exists in the pool, returns it. Otherwise allocates a new entry. Sky flats have their height and lightlevel normalized to 0 so all sky areas share one plane.

**Parameters:**
- `height` - Plane height in fixed-point map units.
- `picnum` - Flat texture number.
- `lightlevel` - Sector light level (0-255).

**Return:** Pointer to an existing or newly created `visplane_t`. Crashes with `I_Error` if the 128-plane limit is reached.

---

### `R_CheckPlane`
```c
visplane_t* R_CheckPlane(visplane_t* pl, int start, int stop)
```
**Purpose:** Validates that visplane `pl` can accept screen columns `[start, stop]` without conflicting with columns already claimed by `pl`. If columns overlap, clones `pl` into a new visplane and returns the clone. If no overlap, extends `pl->minx`/`pl->maxx` and returns `pl`.

**Parameters:**
- `pl` - The visplane to extend or split.
- `start` - First column of the new range.
- `stop` - Last column of the new range.

**Return:** Either `pl` (range was merged in place) or a pointer to a new cloned `visplane_t`.

---

## Data Structures

### `planefunction_t`

```c
typedef void (*planefunction_t)(int top, int bottom);
```

A function pointer type for a plane-drawing callback taking top and bottom pixel boundaries. Defined here for the `floorfunc`/`ceilingfunc_t` pointers, but not used in the Linux 1.10 rendering path.

### `visplane_t` (defined in `r_defs.h`)

Used extensively by the functions declared here. See `r_defs.h` documentation for the full structure layout. Key fields:

| Field | Type | Meaning |
|-------|------|---------|
| `height` | `fixed_t` | Plane height in world units. |
| `picnum` | `int` | Flat lump index. `skyflatnum` = sky plane. |
| `lightlevel` | `int` | Sector brightness. |
| `minx`, `maxx` | `int` | Leftmost and rightmost active columns. |
| `top[SCREENWIDTH]` | `byte[]` | Top pixel row per column (`0xFF` = empty). |
| `bottom[SCREENWIDTH]` | `byte[]` | Bottom pixel row per column. |

---

## Dependencies

| Header | Provides |
|--------|---------|
| `r_data.h` | `lighttable_t`, `visplane_t` (via `r_defs.h`), screen dimension constants. |
