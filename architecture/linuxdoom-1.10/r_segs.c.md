# File Overview

**Path:** `linuxdoom-1.10/r_segs.c`
**Module:** Renderer - Wall Segment (Seg) Rendering

`r_segs.c` is the wall-rendering engine. It converts BSP line segments (segs) into vertical columns of textured pixels in the framebuffer. It is one of the most compute-intensive files in the engine during gameplay.

The module provides two main entry points:

- **`R_StoreWallRange(start, stop)`** - Called by `r_bsp.c` for each seg that passes visibility testing. It performs all per-seg setup (distance, scale, texture selection, plane marking) and then calls `R_RenderSegLoop` to draw the columns.
- **`R_RenderMaskedSegRange(ds, x1, x2)`** - Called by `r_things.c` during the back-to-front masked drawing pass to render transparent mid-textures on two-sided lines (glass, bars, etc.).

### Wall Rendering Model

DOOM walls are rendered as vertical columns, one per screen X coordinate. For each column:

1. A perspective-correct **scale** value is computed. Scale is the ratio of the projection plane distance to the perpendicular wall distance at that column. Larger scale = closer wall = taller column.
2. The scale maps a world-height range to a screen-pixel range. For a solid wall the column spans `[topfrac, bottomfrac]`; for a two-sided line separate ranges apply to the upper texture (step), lower texture (step), and the open gap in the middle.
3. The **texture column** index is derived from the horizontal offset along the wall using `rw_offset - tan(angle_from_normal) * rw_distance`.
4. Lighting is read from `walllights[scale >> LIGHTSCALESHIFT]` - closer (larger scale) columns are brighter.

### One-Sided vs. Two-Sided Lines

A seg with no back sector (`backsector == NULL`) is a **solid wall** (one-sided line). It has:
- A single mid-texture that fills the full wall column.
- No gap for planes to show through.
- Always marks both floor and ceiling.

A seg with a back sector is a **two-sided wall** (portal). It may have:
- An **upper texture** covering the step between the front ceiling and the back ceiling (when the back ceiling is lower).
- A **lower texture** covering the step between the front floor and the back floor (when the back floor is higher).
- A **masked mid-texture** (optional transparent texture like a fence or bars), deferred to `R_RenderMaskedSegRange`.
- No textures at all on either or both bands.

### Clipping Arrays

`floorclip[x]` and `ceilingclip[x]` (maintained in `r_plane.c`) bound the drawable region for each column. After a solid wall is drawn, `ceilingclip[x]` is set to the bottom of the wall (nothing can be drawn above it below the wall) and `floorclip[x]` is set to the top of the wall. This prevents later segs and sprites from overdrawing into solid occluded space.

### Texture Pegging

The `ML_DONTPEGBOTTOM` and `ML_DONTPEGTOP` linedef flags control how textures are anchored:

- Without `ML_DONTPEGBOTTOM`: mid-texture top aligns with the ceiling. With it: bottom aligns with the floor (used for doors so the texture slides with the door).
- Without `ML_DONTPEGTOP`: upper texture bottom aligns with the back ceiling. With it: top aligns with the front ceiling (used for sky transfers).
- Lower texture: without `ML_DONTPEGBOTTOM` the bottom aligns with the front floor; with it the top aligns with the front ceiling.

---

## Global Variables

### Seg State Flags

| Type | Name | Purpose |
|------|------|---------|
| `boolean` | `segtextured` | True if any of the three texture slots (mid, top, bottom) or masked-texture flag are non-zero. Avoids per-column texture/offset calculations when the wall has no textures. |
| `boolean` | `markfloor` | True if this seg should update the floor visplane column data. Set in `R_StoreWallRange` based on sector height differences and position relative to the view plane. |
| `boolean` | `markceiling` | True if this seg should update the ceiling visplane column data. |
| `boolean` | `maskedtexture` | True if the sidedef has a mid-texture on a two-sided line (e.g. a fence). The texture is not drawn immediately; its column indices are saved in `maskedtexturecol[]` for later rendering by `R_RenderMaskedSegRange`. |
| `int` | `toptexture` | Translated texture number for the upper wall band (0 if absent). |
| `int` | `bottomtexture` | Translated texture number for the lower wall band (0 if absent). |
| `int` | `midtexture` | Translated texture number for the solid mid-texture (0 if absent). Non-zero only for one-sided lines. |

### Seg Geometry

| Type | Name | Purpose |
|------|------|---------|
| `angle_t` | `rw_normalangle` | The angle perpendicular to the wall (wall normal). Equals `curline->angle + ANG90`. Used in scale and offset calculations. |
| `int` | `rw_angle1` | Binary angle from the viewpoint to the first vertex of the seg. Used to compute offset angle for distance calculation. |
| `int` | `rw_x` | Current screen X being rendered in `R_RenderSegLoop`. Incremented each iteration. |
| `int` | `rw_stopx` | One past the last X to render (`stop + 1`). Loop condition is `rw_x < rw_stopx`. |
| `angle_t` | `rw_centerangle` | `ANG90 + viewangle - rw_normalangle`. The angular difference between the view direction and the wall normal, used as a base for per-column texture column lookup. |
| `fixed_t` | `rw_offset` | Horizontal texture offset along the wall. Accounts for the seg's starting vertex position, sidedef `textureoffset`, and the `curline->offset`. |
| `fixed_t` | `rw_distance` | Perpendicular distance from the viewpoint to the wall plane. Used in `R_ScaleFromGlobalAngle` and the offset calculation. |
| `fixed_t` | `rw_scale` | Scale at the current column `rw_x`. Updated by `+= rw_scalestep` each column. |
| `fixed_t` | `rw_scalestep` | Per-column scale increment: `(scale2 - scale1) / (stop - start)`. Linear interpolation across the seg. |
| `fixed_t` | `rw_midtexturemid` | Texture-Y coordinate of the mid-texture reference point (adjusted for pegging and rowoffset). |
| `fixed_t` | `rw_toptexturemid` | Texture-Y coordinate of the upper-texture reference point. |
| `fixed_t` | `rw_bottomtexturemid` | Texture-Y coordinate of the lower-texture reference point. |

### World-Space Vertical Extents

| Type | Name | Purpose |
|------|------|---------|
| `int` | `worldtop` | Front sector ceiling height minus viewz (in world units, not yet scaled to screen). |
| `int` | `worldbottom` | Front sector floor height minus viewz. |
| `int` | `worldhigh` | Back sector ceiling height minus viewz (two-sided lines only). |
| `int` | `worldlow` | Back sector floor height minus viewz (two-sided lines only). |

### Screen-Space Fractional Pixel Bounds

These are `fixed_t` values where the integer part is the screen Y pixel and the fraction provides sub-pixel accuracy for the stepped edge:

| Type | Name | Purpose |
|------|------|---------|
| `fixed_t` | `pixhigh` | Current fractional screen Y of the back-sector ceiling edge (top texture bottom boundary). |
| `fixed_t` | `pixlow` | Current fractional screen Y of the back-sector floor edge (bottom texture top boundary). |
| `fixed_t` | `pixhighstep` | Per-column step for `pixhigh`. |
| `fixed_t` | `pixlowstep` | Per-column step for `pixlow`. |
| `fixed_t` | `topfrac` | Current fractional screen Y of the front sector ceiling (wall top edge). |
| `fixed_t` | `topstep` | Per-column step for `topfrac`. |
| `fixed_t` | `bottomfrac` | Current fractional screen Y of the front sector floor (wall bottom edge). |
| `fixed_t` | `bottomstep` | Per-column step for `bottomfrac`. |

### Lighting and Masking

| Type | Name | Purpose |
|------|------|---------|
| `lighttable_t**` | `walllights` | Pointer to a row of `scalelight[lightlevel_index][]`. Indexed by `rw_scale >> LIGHTSCALESHIFT` to get the colormap for each column. Also referenced externally via `extern` in `r_main.c`. |
| `short*` | `maskedtexturecol` | Pointer into the `openings[]` buffer where texture column indices for a masked mid-texture are being stored. Indexed by screen X during `R_RenderSegLoop`; read back during `R_RenderMaskedSegRange`. |

### Local Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `HEIGHTBITS` | 12 | Fractional bits used in the height stepping variables (`topfrac`, `bottomfrac`, etc.). The integer part gives the screen pixel row. |
| `HEIGHTUNIT` | `1<<12 = 4096` | One pixel in the height-stepping fixed-point format. Used to round up when converting `topfrac` to an integer row. |

---

## Functions

### `R_RenderMaskedSegRange`
```c
void R_RenderMaskedSegRange(drawseg_t* ds, int x1, int x2)
```
**Purpose:** Renders the masked mid-texture of a two-sided linedef for columns `x1` to `x2`. Called during the back-to-front masked pass (`R_DrawMasked`) after all opaque geometry is rendered, so transparent textures composite correctly over the walls behind them.

**Parameters:**
- `ds` - The `drawseg_t` record that was created for this seg during the original forward pass. Contains precomputed scale values, clip arrays, and the `maskedtexturecol` array.
- `x1`, `x2` - Column range to render (inclusive).

**Key logic:**

1. Restores `curline`, `frontsector`, `backsector` from `ds->curline`.
2. Determines the light table for the seg using `frontsector->lightlevel` with directional adjustment (horizontal lines get -1, vertical +1 to simulate directional lighting).
3. Copies `maskedtexturecol`, `rw_scalestep`, and initial `spryscale` from the draw-seg.
4. Computes `dc_texturemid` based on the `ML_DONTPEGBOTTOM` flag: if set, the texture bottom anchors to `max(frontsector->floorheight, backsector->floorheight)` plus texture height; otherwise the texture top anchors to `min(frontsector->ceilingheight, backsector->ceilingheight)`. Applies `sidedef->rowoffset`.
5. For each column `dc_x` from `x1` to `x2`:
   - Skips columns where `maskedtexturecol[dc_x] == MAXSHORT` (no texture there).
   - Computes `dc_colormap` from `walllights[spryscale >> LIGHTSCALESHIFT]`.
   - Computes `sprtopscreen = centeryfrac - FixedMul(dc_texturemid, spryscale)`.
   - Computes `dc_iscale = 0xFFFFFFFF / spryscale` (inverse scale for texture Y stepping).
   - Fetches the texture column: `col = R_GetColumn(texnum, maskedtexturecol[dc_x]) - 3` (the `-3` backs up to the `column_t` header).
   - Calls `R_DrawMaskedColumn(col)` to draw the column with top/bottom clipping from `mceilingclip`/`mfloorclip`.
   - Marks `maskedtexturecol[dc_x] = MAXSHORT` to avoid redrawing.
   - Advances `spryscale += rw_scalestep`.

---

### `R_RenderSegLoop`
```c
void R_RenderSegLoop(void)
```
**Purpose:** The core wall-rendering loop. Called by `R_StoreWallRange`. Iterates columns from `rw_x` to `rw_stopx-1`, rendering up to two wall texture bands (top and bottom) per column and marking floor/ceiling visplane data.

**Key logic (per column):**

**1. Compute top pixel `yl`:**
```
yl = (topfrac + HEIGHTUNIT - 1) >> HEIGHTBITS   (ceiling-divide)
yl = max(yl, ceilingclip[rw_x] + 1)             (clip to ceiling)
```

**2. Mark ceiling visplane (if `markceiling`):**
- The ceiling region is `[ceilingclip[rw_x]+1, yl-1]`, clamped to be above `floorclip[rw_x]`.
- Written into `ceilingplane->top[rw_x]` and `ceilingplane->bottom[rw_x]`.

**3. Compute bottom pixel `yh`:**
```
yh = bottomfrac >> HEIGHTBITS
yh = min(yh, floorclip[rw_x] - 1)              (clip to floor)
```

**4. Mark floor visplane (if `markfloor`):**
- The floor region is `[yh+1, floorclip[rw_x]-1]`, clamped to be below `ceilingclip[rw_x]`.
- Written into `floorplane->top[rw_x]` and `floorplane->bottom[rw_x]`.

**5. Texture column and lighting (if `segtextured`):**
```
angle = (rw_centerangle + xtoviewangle[rw_x]) >> ANGLETOFINESHIFT
texturecolumn = rw_offset - FixedMul(finetangent[angle], rw_distance)
texturecolumn >>= FRACBITS
dc_colormap = walllights[rw_scale >> LIGHTSCALESHIFT]
dc_iscale = 0xFFFFFFFF / rw_scale
```

**6. Solid wall (single-sided, `midtexture` != 0):**
- Draws the full column `[yl, yh]` with `midtexture`.
- Sets `ceilingclip[rw_x] = viewheight` and `floorclip[rw_x] = -1` (column is fully occluded).

**7. Two-sided wall (upper and lower textures):**

*Upper texture* (if `toptexture` != 0):
```
mid = pixhigh >> HEIGHTBITS    (where the back ceiling edge is)
pixhigh += pixhighstep
mid = min(mid, floorclip[rw_x] - 1)
if mid >= yl:
    draw column [yl, mid] with toptexture
    ceilingclip[rw_x] = mid
else:
    ceilingclip[rw_x] = yl - 1
```
If no upper texture: `ceilingclip[rw_x] = yl - 1` (when `markceiling`).

*Lower texture* (if `bottomtexture` != 0):
```
mid = (pixlow + HEIGHTUNIT - 1) >> HEIGHTBITS
pixlow += pixlowstep
mid = max(mid, ceilingclip[rw_x] + 1)
if mid <= yh:
    draw column [mid, yh] with bottomtexture
    floorclip[rw_x] = mid
else:
    floorclip[rw_x] = yh + 1
```
If no lower texture: `floorclip[rw_x] = yh + 1` (when `markfloor`).

*Masked texture* (if `maskedtexture`): saves `maskedtexturecol[rw_x] = texturecolumn` for later drawing.

**8. Step:**
```
rw_scale     += rw_scalestep
topfrac      += topstep
bottomfrac   += bottomstep
```

---

### `R_StoreWallRange`
```c
void R_StoreWallRange(int start, int stop)
```
**Purpose:** Sets up all per-seg rendering state and records a `drawseg_t` for a wall segment covering screen columns `[start, stop]`. Called once per seg by `R_ClipPassWallRange` / `R_ClipSolidWallRange` in `r_bsp.c`. After setup, directly calls `R_RenderSegLoop`.

**Parameters:**
- `start` - First screen column covered by this seg (inclusive).
- `stop` - Last screen column covered by this seg (inclusive).

**Detailed steps:**

**1. Guard against overflow:**
Checks `ds_p == &drawsegs[MAXDRAWSEGS]` and returns silently if the 256-entry draw-seg limit is hit.

**2. Mark linedef as mapped:**
Sets `ML_MAPPED` on `linedef->flags` so the automap shows the line.

**3. Compute `rw_distance` (perpendicular wall distance):**
```
rw_normalangle = curline->angle + ANG90
offsetangle = clamp(|rw_normalangle - rw_angle1|, 0, ANG90)
distangle = ANG90 - offsetangle
hyp = R_PointToDist(curline->v1->x, curline->v1->y)
rw_distance = hyp * sin(distangle)
```
This gives the perpendicular (not radial) distance from the viewer to the wall plane, which is what the perspective formula requires.

**4. Compute scales:**
```
ds_p->scale1 = rw_scale = R_ScaleFromGlobalAngle(viewangle + xtoviewangle[start])
ds_p->scale2 = R_ScaleFromGlobalAngle(viewangle + xtoviewangle[stop])
rw_scalestep = (scale2 - scale1) / (stop - start)
```
Scale is computed at both endpoints and linearly interpolated across the seg.

**5. Texture boundary and plane marking decisions:**
- `worldtop = frontsector->ceilingheight - viewz`
- `worldbottom = frontsector->floorheight - viewz`

For one-sided lines: selects `midtexture`; `markfloor = markceiling = true`.

For two-sided lines:
- Computes `worldhigh = backsector->ceilingheight - viewz`, `worldlow = backsector->floorheight - viewz`.
- Sky-hack: if both front and back ceilings are sky, sets `worldtop = worldhigh` (prevents a seam at the sky boundary).
- `markfloor`: true if floor heights differ, or flat or light level differs between sectors.
- `markceiling`: true if ceiling heights differ, or ceiling flat or light level differs.
- Closed door check: if `backsector->ceilingheight <= frontsector->floorheight` or `backsector->floorheight >= frontsector->ceilingheight`, forces both marks true.
- Selects `toptexture` if `worldhigh < worldtop` (back ceiling is lower = upper step visible).
- Selects `bottomtexture` if `worldlow > worldbottom` (back floor is higher = lower step visible).
- Allocates masked texture column array in `openings[]` if `sidedef->midtexture` is set.

**6. View-plane culling:**
- `markfloor = false` if `frontsector->floorheight >= viewz` (floor is above the eye - can't see it).
- `markceiling = false` if `frontsector->ceilingheight <= viewz && ceilingpic != skyflatnum` (ceiling is below the eye).

**7. Compute texture pegging (mid offset):**
Sets `rw_midtexturemid`, `rw_toptexturemid`, `rw_bottomtexturemid` according to the `ML_DONTPEGBOTTOM`/`ML_DONTPEGTOP` flags and `sidedef->rowoffset`.

**8. Silhouette and sprite clip setup:**
Records `ds_p->silhouette` bitmask (`SIL_BOTTOM`, `SIL_TOP`, or both) based on height comparisons. This is used by `R_DrawMasked` to determine which clip arrays a sprite needs to consult for each draw-seg it overlaps.

**9. Compute incremental height stepping values:**
```
worldtop    >>= 4    (shift from fixed-point to HEIGHTBITS format)
worldbottom >>= 4
topstep    = -FixedMul(rw_scalestep, worldtop)
topfrac    = (centeryfrac >> 4) - FixedMul(worldtop, rw_scale)
bottomstep = -FixedMul(rw_scalestep, worldbottom)
bottomfrac = (centeryfrac >> 4) - FixedMul(worldbottom, rw_scale)
```
`pixhigh`/`pixhighstep` and `pixlow`/`pixlowstep` are computed similarly for the two-sided edge pixels.

**10. Register planes:**
```
if markceiling: ceilingplane = R_CheckPlane(ceilingplane, rw_x, rw_stopx-1)
if markfloor:   floorplane   = R_CheckPlane(floorplane, rw_x, rw_stopx-1)
```

**11. Render:**
Calls `R_RenderSegLoop()`.

**12. Save sprite clip arrays:**
After the seg loop, if the draw-seg needs `sprtopclip` or `sprbottomclip` (for sprite clipping against this wall), copies `ceilingclip` or `floorclip` into the `openings[]` buffer and records the pointer in `ds_p`. Advances `lastopening`.

**13. Advance draw-seg pointer:**
`ds_p++`.

**Texture coordinate formula (within `R_RenderSegLoop`):**
```
angle = (rw_centerangle + xtoviewangle[rw_x]) >> ANGLETOFINESHIFT
texturecolumn = (rw_offset - FixedMul(finetangent[angle], rw_distance)) >> FRACBITS
```

---

## Data Structures

No new structs are defined in this file. The following structures from `r_defs.h` are central to its operation:

### `drawseg_t`

```c
typedef struct drawseg_s {
    seg_t*   curline;         // The seg being drawn
    int      x1, x2;          // Screen column range
    fixed_t  scale1, scale2;  // Scale at leftmost and rightmost columns
    fixed_t  scalestep;       // Per-column scale increment
    int      silhouette;      // SIL_NONE/BOTTOM/TOP/BOTH
    fixed_t  bsilheight;      // Sprite clipping: do not clip below this height
    fixed_t  tsilheight;      // Sprite clipping: do not clip above this height
    short*   sprtopclip;      // Per-column ceiling clip for sprites
    short*   sprbottomclip;   // Per-column floor clip for sprites
    short*   maskedtexturecol;// Per-column texture column for masked mid-texture
} drawseg_t;
```

One `drawseg_t` is filled for every seg that `R_StoreWallRange` processes. The array `drawsegs[MAXDRAWSEGS]` (max 256 entries) is defined in `r_bsp.c`. `ds_p` advances through it during BSP traversal and is read back by `R_DrawMasked` in `r_things.c`.

### `seg_t`

```c
typedef struct {
    vertex_t* v1, *v2;      // World-space endpoints
    fixed_t   offset;        // Texture start offset along the wall
    angle_t   angle;         // Angle of the seg (v1->v2 direction)
    side_t*   sidedef;       // Wall textures and offsets
    line_t*   linedef;       // Parent linedef (flags, sector refs)
    sector_t* frontsector;   // Sector on the front side
    sector_t* backsector;    // Sector on the back side (NULL if one-sided)
} seg_t;
```

### `side_t`

```c
typedef struct {
    fixed_t textureoffset;   // Horizontal texture offset (added to rw_offset)
    fixed_t rowoffset;       // Vertical texture offset (added to texture mids)
    short   toptexture;      // Upper wall texture index
    short   bottomtexture;   // Lower wall texture index
    short   midtexture;      // Mid wall texture index (masked on 2-sided)
    sector_t* sector;        // The sector this side faces
} side_t;
```

---

## Dependencies

| Module | Usage |
|--------|-------|
| `r_local.h` | Master renderer header; provides `curline`, `frontsector`, `backsector`, `sidedef`, `linedef`, `drawsegs`, `ds_p`, `dc_*` column globals, `ds_*` span globals (unused here), all other renderer globals. |
| `r_sky.h` | `skyflatnum` for sky-hack ceiling check. |
| `r_main.h` | `viewangle`, `viewz`, `xtoviewangle[]`, `centeryfrac`, `fixedcolormap`, `detailshift`, `colfunc`, `scalelight[][]`, `walllights`, `extralight`, `LIGHTLEVELS`, `LIGHTSCALESHIFT`, `R_ScaleFromGlobalAngle`, `R_PointToDist`. |
| `r_plane.h` | `floorclip[]`, `ceilingclip[]`, `floorplane`, `ceilingplane`, `R_FindPlane`, `R_CheckPlane`, `lastopening`, `openings[]`. |
| `r_data.h` | `R_GetColumn()`, `texturetranslation[]`, `textureheight[]`, `colormaps`. |
| `r_things.h` | `negonearray[]`, `screenheightarray[]`, `spryscale`, `sprtopscreen`, `mfloorclip`, `mceilingclip`, `R_DrawMaskedColumn`. |
| `r_draw.h` | `R_DrawColumn`, `dc_x`, `dc_yl`, `dc_yh`, `dc_iscale`, `dc_texturemid`, `dc_source`, `dc_colormap`. |
| `tables.h` | `finesine[]`, `finetangent[]`, `ANG90`, `ANG180`, `ANGLETOFINESHIFT`. |
| `doomdef.h` | `FRACBITS`, `FRACUNIT`, `fixed_t`, `angle_t`, `boolean`, `MAXSHORT`, `MAXINT`, `MININT`. |
| `doomstat.h` | `SIL_NONE`, `SIL_BOTTOM`, `SIL_TOP`, `SIL_BOTH`, `ML_MAPPED`, `ML_DONTPEGBOTTOM`, `ML_DONTPEGTOP`, `MAXDRAWSEGS`. |
| `i_system.h` | `I_Error()` for range-check failures. |
