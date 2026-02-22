# File Overview

**Path:** `linuxdoom-1.10/r_plane.c`
**Module:** Renderer - Floor and Ceiling (Plane) Rendering

`r_plane.c` is responsible for rendering horizontal flat surfaces - floors and ceilings - as well as the sky. It operates in two distinct phases that match the two phases of the overall rendering pipeline:

**Phase 1 (Accumulation - during BSP traversal):** As `R_RenderBSPNode` processes wall segs from front to back, `R_StoreWallRange` calls `R_FindPlane` and `R_CheckPlane` to record which floor and ceiling surfaces are visible and in which screen columns. Each unique `(height, texture, lightlevel)` combination produces a `visplane_t`. For each screen column where a plane is visible, the top and bottom pixel rows of that column's plane area are stored in `visplane_t::top[]` and `visplane_t::bottom[]`.

**Phase 2 (Drawing - after BSP traversal):** `R_DrawPlanes` iterates all accumulated visplanes. For floors and ceilings it calls `R_MakeSpans`, which converts the per-column top/bottom data into horizontal spans and calls `R_MapPlane` on each run of consecutive columns sharing the same vertical extent. The sky is a special case drawn as vertical columns instead.

### The Visplane Merging Problem

DOOM's key space optimization is "visplane merging." Every sector visible through an opening could in theory produce a separate visplane entry per screen column. Merging avoids an explosion in the number of visplanes:

- `R_FindPlane` searches the existing `visplanes[]` array for a plane with matching `height`, `picnum`, and `lightlevel`. If found, it returns the existing entry.
- `R_CheckPlane` is called when a segs's column range overlaps a plane that was already used for other columns. It checks whether the new range is disjoint from the already-claimed columns. If disjoint, it merges (extends `minx`/`maxx`). If overlapping (a column is claimed twice), it allocates a new visplane with the same properties.

The practical limit `MAXVISPLANES = 128` means overly complex scenes with many distinct planes at different heights or light levels can overflow and crash. This is a known limitation of the original engine.

### Span Rasterization

Each visplane's column data is a sparse array of `(top, bottom)` pairs indexed by screen X. `R_DrawPlanes` scans left-to-right and calls `R_MakeSpans` at each column transition. `R_MakeSpans` uses a simple "open spans" model:

- `spanstart[y]` records the X at which a span at row `y` began.
- When a row that was "open" in the previous column is no longer open in the current column, the span `[spanstart[y], x-1]` is complete and `R_MapPlane` is called.
- When a row becomes open that was not open, `spanstart[y]` is set to the current X.

### Floor/Ceiling Texture Mapping

Floors and ceilings are flat-textured planes in 3D space. To texture-map them DOOM uses a perspective-correct formula for each horizontal span:

```
distance = planeheight * yslope[y]
ds_xstep = distance * basexscale
ds_ystep = distance * baseyscale
length   = distance * distscale[x1]
angle    = viewangle + xtoviewangle[x1]
ds_xfrac = viewx + cos(angle) * length
ds_yfrac = -viewy - sin(angle) * length
```

`yslope[y]` pre-computes the distance scale for each screen row (computed in `R_ExecuteSetViewSize`). `distscale[x]` corrects for the oblique angle of the span relative to the view direction.

Results are cached per row (`cachedheight[]`, `cacheddistance[]`, `cachedxstep[]`, `cachedystep[]`) since all spans at the same screen row and same plane height have identical distance/step values regardless of X position.

---

## Global Variables

| Type | Name | Purpose |
|------|------|---------|
| `planefunction_t` | `floorfunc` | Declared but not used in the Linux source; was intended as a function pointer for floor drawing dispatch. |
| `planefunction_t` | `ceilingfunc` | Declared but not used in the Linux source; same purpose as `floorfunc`. |
| `visplane_t` | `visplanes[MAXVISPLANES]` | Pool of 128 visplane records. Allocated linearly from index 0 upward each frame. |
| `visplane_t*` | `lastvisplane` | Points to the next free slot in `visplanes[]`. Reset to `visplanes` by `R_ClearPlanes`. Used as a stack-allocator pointer. |
| `visplane_t*` | `floorplane` | Pointer into `visplanes[]` for the current subsector's floor plane. Set by `R_StoreWallRange` via `R_FindPlane`. |
| `visplane_t*` | `ceilingplane` | Pointer into `visplanes[]` for the current subsector's ceiling plane. Set by `R_StoreWallRange` via `R_FindPlane`. |
| `short` | `openings[MAXOPENINGS]` | Flat buffer (`SCREENWIDTH * 64 = 20480` entries) used for storing sprite clip arrays (`sprtopclip`, `sprbottomclip`) and masked texture column indices. `lastopening` advances through it each frame. |
| `short*` | `lastopening` | Next free position in `openings[]`. Reset to `openings` each frame. |
| `short` | `floorclip[SCREENWIDTH]` | Per-column floor clip. Initialized to `viewheight` each frame. Decremented as solid wall segments fill upward from the bottom. Spans are only drawn above this line. |
| `short` | `ceilingclip[SCREENWIDTH]` | Per-column ceiling clip. Initialized to `-1` each frame. Incremented as solid wall segments fill downward from the top. Spans are only drawn below this line. |
| `int` | `spanstart[SCREENHEIGHT]` | Per-row X coordinate at which the currently-open span began. Written by `R_MakeSpans` when a span opens; read when the span closes. |
| `int` | `spanstop[SCREENHEIGHT]` | Declared but unused in the Linux 1.10 source. Was presumably intended as a parallel end-position array. |
| `lighttable_t**` | `planezlight` | Pointer to a row of `zlight[][]` for the plane currently being drawn. Set from `zlight[lightlevel]` before the span loop. |
| `fixed_t` | `planeheight` | Absolute height difference `|plane->height - viewz|` for the plane being drawn. Used in the distance formula. |
| `fixed_t` | `yslope[SCREENHEIGHT]` | Precomputed per-row slope factor: `(viewwidth/2 * FRACUNIT) / |row - centery|`. Larger near the horizon (center), zero at the very center row. Computed in `R_ExecuteSetViewSize`. |
| `fixed_t` | `distscale[SCREENWIDTH]` | Precomputed per-column `1/cos(xtoviewangle[x])` factor. Corrects for the oblique view angle when computing floor-span distances. Computed in `R_ExecuteSetViewSize`. |
| `fixed_t` | `basexscale` | X texture step base: `cos(viewangle-90) / centerxfrac`. Multiplied by distance to get `ds_xstep`. Recomputed each frame in `R_ClearPlanes`. |
| `fixed_t` | `baseyscale` | Y texture step base: `-sin(viewangle-90) / centerxfrac`. Multiplied by distance to get `ds_ystep`. Recomputed each frame in `R_ClearPlanes`. |
| `fixed_t` | `cachedheight[SCREENHEIGHT]` | Cache key: the `planeheight` value for which the cached row values were computed. |
| `fixed_t` | `cacheddistance[SCREENHEIGHT]` | Cached `distance = planeheight * yslope[y]` per row. |
| `fixed_t` | `cachedxstep[SCREENHEIGHT]` | Cached `ds_xstep = distance * basexscale` per row. |
| `fixed_t` | `cachedystep[SCREENHEIGHT]` | Cached `ds_ystep = distance * baseyscale` per row. |

### Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `MAXVISPLANES` | 128 | Maximum number of distinct visplanes per frame. Overflow causes `I_Error`. |
| `MAXOPENINGS` | `SCREENWIDTH*64` | Size of the `openings[]` scratch buffer in `short` elements. |

---

## Functions

### `R_InitPlanes`
```c
void R_InitPlanes(void)
```
**Purpose:** Startup initialization for the plane subsystem. Currently a stub with no body - all per-frame setup happens in `R_ClearPlanes`. Called once from `R_Init`.

---

### `R_MapPlane`
```c
void R_MapPlane(int y, int x1, int x2)
```
**Purpose:** The innermost floor/ceiling rendering primitive. Renders a single horizontal span from pixel `(x1, y)` to `(x2, y)` for the currently active flat texture. Sets all span-drawing globals and calls `spanfunc`.

**Parameters:**
- `y` - Screen row of the span.
- `x1` - Leftmost pixel column (inclusive).
- `x2` - Rightmost pixel column (inclusive).

**Key logic:**

1. **Distance/step caching:** If `planeheight != cachedheight[y]`, recomputes:
   - `distance = FixedMul(planeheight, yslope[y])` - how far away this row is.
   - `ds_xstep = FixedMul(distance, basexscale)` - texture X increment per screen pixel.
   - `ds_ystep = FixedMul(distance, baseyscale)` - texture Y increment per screen pixel.
   - Stores all three in the cache arrays for this row.

2. **Start position:** Computes the texture coordinates of the leftmost span pixel:
   - `length = FixedMul(distance, distscale[x1])` - actual world distance to the floor point at column `x1`.
   - `angle = (viewangle + xtoviewangle[x1]) >> ANGLETOFINESHIFT` - exact view direction at `x1`.
   - `ds_xfrac = viewx + cos(angle) * length` - world X mapped into texture space.
   - `ds_yfrac = -viewy - sin(angle) * length` - world Y mapped into texture space (negated for DOOM's coordinate convention).

3. **Lighting:** If `fixedcolormap` is active, uses it directly. Otherwise indexes `planezlight[distance >> LIGHTZSHIFT]`.

4. Sets `ds_y`, `ds_x1`, `ds_x2` and calls `spanfunc()`.

**Globals read:** `planeheight`, `basexscale`, `baseyscale`, `viewx`, `viewy`, `viewangle`, `yslope[]`, `distscale[]`, `xtoviewangle[]`, `planezlight`, `fixedcolormap`.

**Globals written:** `ds_xstep`, `ds_ystep`, `ds_xfrac`, `ds_yfrac`, `ds_colormap`, `ds_y`, `ds_x1`, `ds_x2`, cache arrays.

---

### `R_ClearPlanes`
```c
void R_ClearPlanes(void)
```
**Purpose:** Per-frame reset of the entire plane subsystem. Called at the start of each frame by `R_RenderPlayerView`.

**What it does:**
1. Sets `floorclip[x] = viewheight` and `ceilingclip[x] = -1` for all columns. These represent the full unclipped state (floor can extend from any row up to the bottom of the screen; ceiling from any row down to the top).
2. Resets `lastvisplane = visplanes` (discards all visplanes from the previous frame).
3. Resets `lastopening = openings`.
4. Zeros `cachedheight[]` to invalidate the distance cache.
5. Computes `basexscale` and `baseyscale` from the current `viewangle`:
   ```
   angle = (viewangle - ANG90) >> ANGLETOFINESHIFT
   basexscale =  FixedDiv(finecosine[angle], centerxfrac)
   baseyscale = -FixedDiv(finesine[angle],   centerxfrac)
   ```
   These give the world-space texture step per unit of screen distance for the left-right direction perpendicular to the view.

---

### `R_FindPlane`
```c
visplane_t* R_FindPlane(fixed_t height, int picnum, int lightlevel)
```
**Purpose:** Finds or creates a visplane for a surface with the given `height`, texture `picnum`, and `lightlevel`. Called by `R_StoreWallRange` for each subsector's floor and ceiling.

**Key logic:**
- Sky flats are special-cased: height and lightlevel are both forced to 0 so all sky areas share a single visplane regardless of position.
- Scans `visplanes[0..lastvisplane-1]` for an exact match on all three fields. Returns the matching entry immediately if found.
- If not found, allocates a new entry at `lastvisplane++`. Initializes `minx = SCREENWIDTH`, `maxx = -1`, and fills `top[]` with `0xFF` (sentinel meaning "this column not yet assigned to this plane").
- Calls `I_Error` if `MAXVISPLANES` (128) is exceeded.

**Parameters:**
- `height` - Floor or ceiling height in map units (16.16 fixed-point), relative to the view z is applied later.
- `picnum` - Flat lump number (index into `flattranslation[]`).
- `lightlevel` - Sector light level (0-255).

**Return:** Pointer to the matching or newly allocated `visplane_t`.

---

### `R_CheckPlane`
```c
visplane_t* R_CheckPlane(visplane_t* pl, int start, int stop)
```
**Purpose:** Verifies that visplane `pl` can accept the screen column range `[start, stop]` without conflict. If the range overlaps columns already claimed by `pl`, a new visplane is created as a clone with the same properties. Called from `R_StoreWallRange` immediately before `R_RenderSegLoop`.

**Key logic:**
1. Computes the intersection `[intrl, intrh]` of `[start,stop]` with `[pl->minx, pl->maxx]`.
2. Scans the intersection for any column where `pl->top[x] != 0xFF`. If such a column exists, the plane is already "occupied" in that range and must be split.
3. If no overlap conflict exists: extends `pl->minx`/`pl->maxx` to the union and returns `pl`.
4. If conflict: clones `pl->height`, `pl->picnum`, `pl->lightlevel` into a new `lastvisplane++` entry; sets new `minx = start`, `maxx = stop`, clears `top[]` to `0xFF`; returns the new plane.

**Parameters:**
- `pl` - Existing visplane to check/extend.
- `start`, `stop` - The column range this seg will contribute.

**Return:** Either `pl` (if extended in place) or a pointer to a new clone visplane.

---

### `R_MakeSpans`
```c
void R_MakeSpans(int x, int t1, int b1, int t2, int b2)
```
**Purpose:** Manages the transitions in the "open spans" state machine as the drawing loop moves from column `x-1` to column `x`. Closes spans that are no longer open and opens spans that are newly visible. Called by `R_DrawPlanes` for each column.

**Parameters:**
- `x` - Current screen column (the new column arriving).
- `t1`, `b1` - Top and bottom of the plane region in the **previous** column (column `x-1`).
- `t2`, `b2` - Top and bottom of the plane region in the **current** column (column `x`).

**Key logic (four loops):**
1. While `t1 < t2` and rows are still valid: rows `[t1..min(b1,t2-1)]` were open last column but not this column. Close them by calling `R_MapPlane(t1, spanstart[t1], x-1)`.
2. While `b1 > b2` and rows are valid: rows `[max(t1,b2+1)..b1]` are closing at the bottom. Call `R_MapPlane(b1, spanstart[b1], x-1)`.
3. While `t2 < t1` and in range: rows `[t2..min(b2,t1-1)]` are newly opening at the top. Set `spanstart[t2] = x`.
4. While `b2 > b1` and in range: rows `[max(t2,b1+1)..b2]` are newly opening at the bottom. Set `spanstart[b2] = x`.

This produces minimal, maximally-long horizontal spans for efficient span rendering.

---

### `R_DrawPlanes`
```c
void R_DrawPlanes(void)
```
**Purpose:** The entry point for the second rendering phase: rasterizes all accumulated visplanes into the framebuffer. Called once per frame from `R_RenderPlayerView` after `R_RenderBSPNode` completes.

**Key logic:**

Iterates every visplane from `visplanes[0]` to `lastvisplane-1`.

**Sky planes:**
- Sets `dc_iscale = pspriteiscale >> detailshift`.
- Sets `dc_colormap = colormaps` (index 0, full bright - sky is never darkened).
- Sets `dc_texturemid = skytexturemid`.
- For each column `x` in `[pl->minx, pl->maxx]`, reads `dc_yl = pl->top[x]`, `dc_yh = pl->bottom[x]`. If the range is valid, computes `angle = (viewangle + xtoviewangle[x]) >> ANGLETOSKYSHIFT` to get the sky texture column, then calls `colfunc()`.

**Floor/ceiling planes:**
- Loads the flat texture data via `W_CacheLumpNum`.
- Computes `planeheight = |pl->height - viewz|`.
- Selects `planezlight` from `zlight[lightlevel_index]`.
- Places sentinel values at `pl->top[pl->minx-1]` and `pl->top[pl->maxx+1]` (set to `0xFF`) to ensure the span machine closes all open spans cleanly at the edges.
- Runs the span machine: for each column `x` from `pl->minx` to `pl->maxx+1`, calls `R_MakeSpans(x, pl->top[x-1], pl->bottom[x-1], pl->top[x], pl->bottom[x])`.
- Releases the flat lump with `Z_ChangeTag(PU_CACHE)`.

---

## Data Structures

### `visplane_t` (defined in `r_defs.h`)

The core data structure for this module.

```c
typedef struct {
    fixed_t  height;              // Floor/ceiling height in map units
    int      picnum;              // Flat texture number
    int      lightlevel;          // Sector light level (0-255)
    int      minx;                // Leftmost column with data (SCREENWIDTH if none)
    int      maxx;                // Rightmost column with data (-1 if none)

    byte     pad1;
    byte     top[SCREENWIDTH];    // Per-column top pixel row (0xFF = empty)
    byte     pad2;
    byte     pad3;
    byte     bottom[SCREENWIDTH]; // Per-column bottom pixel row
    byte     pad4;
} visplane_t;
```

**Field details:**
- `height` - The absolute world-space height of the plane. Used to compute `planeheight = |height - viewz|` during drawing.
- `picnum` - Index into the flat list. If equal to `skyflatnum`, the plane is drawn as a sky column instead of a floor/ceiling span.
- `lightlevel` - Raw sector light value 0-255. Right-shifted by `LIGHTSEGSHIFT` (4) to index `zlight[][]`.
- `minx`, `maxx` - Column range bounding box. Avoids scanning the full `top[]`/`bottom[]` arrays.
- `top[]`/`bottom[]` - Sparse per-column height data. Initialized to `0xFF` (all bytes set to 255, used as "unassigned" sentinel). Written by `R_RenderSegLoop` when marking a floor or ceiling region.
- `pad1`-`pad4` - One-byte pads surrounding `top[]` and `bottom[]` so that the sentinel writes at `top[minx-1]` and `top[maxx+1]` in `R_DrawPlanes` do not read/write out of bounds when `minx=0` or `maxx=SCREENWIDTH-1`.

### `planefunction_t` (defined in `r_plane.h`)

```c
typedef void (*planefunction_t)(int top, int bottom);
```

A function pointer type for plane-drawing dispatch. Declared in `r_plane.h` but not meaningfully used in the Linux 1.10 release; `floorfunc` and `ceilingfunc` are defined but never called.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `r_local.h` | Pulls in all renderer headers (`r_main.h`, `r_draw.h`, `r_data.h`, etc.). Provides `ds_*` span globals, `dc_*` column globals, `colormaps`, `xtoviewangle[]`, `finesine[]`, `finecosine[]`, `texturetranslation[]`, `firstflat`, `flattranslation[]`. |
| `r_sky.h` | `skyflatnum`, `skytexture`, `skytexturemid`, `pspriteiscale`. |
| `r_data.h` / `r_data.c` | `R_GetColumn()` for sky texture columns. |
| `r_main.h` | `viewx`, `viewy`, `viewz`, `viewangle`, `projection`, `centerxfrac`, `yslope[]`, `distscale[]`, `xtoviewangle[]`, `fixedcolormap`, `detailshift`, `spanfunc`, `colfunc`, `zlight[][]`, `planezlight`, `LIGHTLEVELS`, `LIGHTZSHIFT`. |
| `r_bsp.h` / `r_segs.c` | `R_StoreWallRange` calls `R_FindPlane`, `R_CheckPlane`, and marks `floorplane->top[]`/`ceilingplane->top[]` via `R_RenderSegLoop`. `drawsegs`, `ds_p`, `MAXDRAWSEGS` checked in `RANGECHECK` block. |
| `doomdef.h` | `SCREENWIDTH`, `SCREENHEIGHT`, `fixed_t`, `boolean`. |
| `doomstat.h` | Game-state globals. |
| `i_system.h` | `I_Error()` for overflow detection. |
| `z_zone.h` | `Z_ChangeTag()` to release flat texture data back to cache after drawing. |
| `w_wad.h` | `W_CacheLumpNum()` to load flat texture data from the WAD. |
