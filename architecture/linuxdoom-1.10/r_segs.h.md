# r_segs.h

**Source file:** `linuxdoom-1.10/r_segs.h`
**Implementation:** `linuxdoom-1.10/r_segs.c`

---

## File Overview

`r_segs.h` is the public interface for DOOM's wall segment rendering module. It belongs to the refresh (R_*) subsystem, which is DOOM's software 3D renderer. The file exposes a single function that handles the deferred drawing of masked (transparent/translucent) mid-textures on two-sided linedefs.

### Role in the renderer

DOOM renders a frame in two major passes:

1. **Solid geometry pass** - The BSP tree (`r_bsp.c`) is traversed front-to-back. For each visible wall segment, `R_StoreWallRange` (internal to `r_segs.c`) is called. This function renders opaque wall tiers (upper, lower, and single-sided mid textures) immediately and records any masked mid-texture columns into a `drawseg_t` entry for later. It also marks the floor and ceiling planes that need to be filled.

2. **Sprite and masked texture pass** - After all solid geometry is drawn and sprites are sorted, `R_DrawMaskedSegRange` (declared in this header) is called by `r_things.c` to go back and paint the deferred masked mid-texture columns over the already-rendered scene, correctly depth-clipped against sprites and other geometry.

The separation between the two passes is necessary because masked textures on two-sided lines (e.g., a metal grating you can see through, or a window) must be drawn in painter's-algorithm order interleaved with sprites, not simply front-to-back.

### Internal functions (not exported by this header)

`r_segs.c` also contains two functions that are used exclusively within the module and are not declared in this header:

- `R_RenderSegLoop` - The per-column core loop that renders wall tiers and marks floor/ceiling span extents for one wall segment's screen-column range. It is called from `R_StoreWallRange`.
- `R_StoreWallRange` - Called by `r_bsp.c` (via `R_AddLine`) when a wall segment is determined to be visible. Computes all rendering parameters, renders opaque tiers immediately via `R_RenderSegLoop`, and stores sprite-clipping information into the `drawsegs` array.

---

## Global Variables

`r_segs.h` declares no `extern` variables. All module-level state variables defined in `r_segs.c` are either exposed through `r_bsp.h` (e.g., `segtextured`, `markfloor`, `markceiling`) or kept as file-scope statics/globals internal to the module.

The following variables are defined in `r_segs.c` and used internally across `R_StoreWallRange`, `R_RenderSegLoop`, and `R_RenderMaskedSegRange`. They are not declared in any header and are therefore private to the module:

| Variable | Type | Description |
|---|---|---|
| `segtextured` | `boolean` | True if any texture on the current segment is potentially visible. |
| `markfloor` | `boolean` | True if the floor visplane needs to be updated for this segment. |
| `markceiling` | `boolean` | True if the ceiling visplane needs to be updated for this segment. |
| `maskedtexture` | `boolean` | True if the two-sided line has a masked mid-texture that must be deferred. |
| `toptexture` | `int` | Texture number for the upper wall tier (two-sided lines only). |
| `bottomtexture` | `int` | Texture number for the lower wall tier (two-sided lines only). |
| `midtexture` | `int` | Texture number for the middle (full-height) wall tier (one-sided lines). |
| `rw_normalangle` | `angle_t` | The wall's surface normal angle (wall angle + 90 degrees). |
| `rw_angle1` | `int` | Angle from the viewpoint to the first vertex of the segment. |
| `rw_x` | `int` | Current screen column being rendered; advances across the segment range. |
| `rw_stopx` | `int` | One past the last screen column for this segment (`x2 + 1`). |
| `rw_centerangle` | `angle_t` | `ANG90 + viewangle - rw_normalangle`; used for texture column offset calculation. |
| `rw_offset` | `fixed_t` | Horizontal texture offset applied to all columns of the segment. |
| `rw_distance` | `fixed_t` | Perpendicular distance from the viewpoint to the wall plane. |
| `rw_scale` | `fixed_t` | Current column scale factor (perspective-corrected). |
| `rw_scalestep` | `fixed_t` | Per-column increment to `rw_scale` across the segment. |
| `rw_midtexturemid` | `fixed_t` | Vertical texture alignment reference for the mid texture. |
| `rw_toptexturemid` | `fixed_t` | Vertical texture alignment reference for the top texture. |
| `rw_bottomtexturemid` | `fixed_t` | Vertical texture alignment reference for the bottom texture. |
| `worldtop` | `int` | Front sector ceiling height relative to view height (in world units). |
| `worldbottom` | `int` | Front sector floor height relative to view height (in world units). |
| `worldhigh` | `int` | Back sector ceiling height relative to view height (two-sided only). |
| `worldlow` | `int` | Back sector floor height relative to view height (two-sided only). |
| `pixhigh` | `fixed_t` | Screen-space top of the upper texture, as a fixed-point fractional pixel row. |
| `pixlow` | `fixed_t` | Screen-space bottom of the lower texture, as a fixed-point fractional pixel row. |
| `pixhighstep` | `fixed_t` | Per-column step for `pixhigh`. |
| `pixlowstep` | `fixed_t` | Per-column step for `pixlow`. |
| `topfrac` | `fixed_t` | Top edge of the wall opening in fractional pixel coordinates. |
| `topstep` | `fixed_t` | Per-column step for `topfrac`. |
| `bottomfrac` | `fixed_t` | Bottom edge of the wall opening in fractional pixel coordinates. |
| `bottomstep` | `fixed_t` | Per-column step for `bottomfrac`. |
| `walllights` | `lighttable_t**` | Pointer into the `scalelight` table selecting the correct light level row for the current wall's sector and orientation. |
| `maskedtexturecol` | `short*` | Per-column texture column indices for the deferred masked mid-texture. Set to `MAXSHORT` for columns that have been drawn. Stored as a pointer offset into the `openings` scratch buffer. |

---

## Functions

### `R_RenderMaskedSegRange`

```c
void R_RenderMaskedSegRange(drawseg_t* ds, int x1, int x2);
```

**Purpose**

Draws the masked (transparent) mid-texture columns for a two-sided linedef that were deferred during the solid geometry pass. This function is called during the sprite rendering phase in `r_things.c` (`R_DrawMasked`), after all opaque walls have been drawn and visible sprites have been sorted by depth.

A masked mid-texture is one that sits on a two-sided line (a line with both a front and back sector) and uses a patch-format texture that includes transparent holes. Common examples include decorative bars, fences, windows, and force fields. Because these textures can be seen through, they cannot be drawn during the opaque front-to-back pass - they must be composited in correct depth order alongside sprites.

**How it works**

1. Restores rendering context from the `drawseg_t` record: the original `curline`, its `frontsector` and `backsector`, and the texture number for the mid-texture.
2. Recomputes the wall light level based on the front sector's `lightlevel`, adjusted for horizontal/vertical wall orientation (a subtle shading effect: horizontal walls are slightly darker, vertical walls slightly lighter).
3. Recalculates the texture vertical alignment (`dc_texturemid`) using the `ML_DONTPEGBOTTOM` linedef flag to determine whether the texture anchors to the floor or the ceiling of the higher of the two sector boundaries.
4. Iterates over each screen column from `x1` to `x2`. For each column where `maskedtexturecol[dc_x]` is not `MAXSHORT` (meaning a masked pixel is due at this column):
   - Selects the appropriate colormap entry from `walllights` based on the current column's scale (`spryscale`), unless a `fixedcolormap` is active (e.g., the player has the light-amplification visor powerup).
   - Computes `sprtopscreen` (the screen Y coordinate of the texture top for this column) and `dc_iscale` (the reciprocal of scale, used for texture row stepping).
   - Retrieves the texture column data via `R_GetColumn`, offset by -3 bytes to point at the `post_t` header preceding the pixel data.
   - Calls `R_DrawMaskedColumn` to render the column through the existing floor/ceiling clip arrays (`mfloorclip`, `mceilingclip`), which were saved into the `drawseg_t` by `R_StoreWallRange`.
   - Marks the column as consumed by setting `maskedtexturecol[dc_x] = MAXSHORT`.
5. Advances `spryscale` by `rw_scalestep` each column.

**Parameters**

| Parameter | Type | Description |
|---|---|---|
| `ds` | `drawseg_t*` | Pointer to the draw segment record produced by `R_StoreWallRange` during the solid pass. Contains all saved rendering state for this wall segment, including the scale values, sprite clip arrays, masked texture column table, and the originating `seg_t`. |
| `x1` | `int` | The first screen column (inclusive) to draw masked texture columns for. May be narrower than `ds->x1` when the function is called for a partial range (e.g., only the portion of the wall not occluded by a sprite). |
| `x2` | `int` | The last screen column (inclusive) to draw masked texture columns for. |

**Return value**

`void`. Results are written directly to the framebuffer via `R_DrawMaskedColumn`.

**Called from**

- `r_things.c`: `R_DrawMasked` - during sprite rendering, once for each `drawseg_t` that has a masked mid-texture, to interleave the texture with sprites at the correct depth.
  - Called at line 892 with a clipped range `[r1, r2]` (the portion of the wall overlapping a given sprite's screen extent).
  - Called at line 980 with the full segment range `[ds->x1, ds->x2]` for segments not intersecting any sprites.

---

## Data Structures

`r_segs.h` does not define any data structures. The key type used by the exported function, `drawseg_t`, is defined in `r_defs.h`.

### `drawseg_t` (defined in `r_defs.h`, used here)

```c
typedef struct drawseg_s
{
    seg_t*    curline;          // The map line segment being rendered
    int       x1;               // First screen column of this segment
    int       x2;               // Last screen column of this segment
    fixed_t   scale1;           // Scale at x1 (perspective factor)
    fixed_t   scale2;           // Scale at x2 (perspective factor)
    fixed_t   scalestep;        // Linear scale interpolation step per column
    int       silhouette;       // SIL_NONE/SIL_BOTTOM/SIL_TOP/SIL_BOTH
    fixed_t   bsilheight;       // Bottom silhouette clip height for sprites
    fixed_t   tsilheight;       // Top silhouette clip height for sprites
    short*    sprtopclip;       // Per-column ceiling clip array for sprites
    short*    sprbottomclip;    // Per-column floor clip array for sprites
    short*    maskedtexturecol; // Per-column texture column indices for masked mid-texture
} drawseg_t;
```

The `maskedtexturecol` field is a pointer into the shared `openings` scratch buffer (managed in `r_bsp.c`). Its indexing is adjusted so that `maskedtexturecol[x1]` is the first valid entry. Each entry holds the texture column index to use when drawing the masked mid-texture at that screen column, or `MAXSHORT` if that column has already been rendered or has no masked pixel.

`sprtopclip` and `sprbottomclip` are similarly offset arrays that provide the per-column vertical clip bounds. These are used by `R_DrawMaskedColumn` during the masked rendering pass to prevent the texture from drawing into floor or ceiling areas.

---

## Dependencies

### Direct includes

`r_segs.h` has no `#include` directives of its own. It relies on the including translation unit to have already included the necessary type definitions before this header is processed. In practice, `r_segs.h` is always included via `r_local.h`, which provides all required types.

### Types required by the function signature

| Type | Defined in | Description |
|---|---|---|
| `drawseg_t` | `r_defs.h` | The draw segment record produced during the solid geometry pass. |
| `int` | C standard library | Screen column indices. |

### Modules `r_segs.c` depends on at implementation level

| Module | Header | Role |
|---|---|---|
| System interface | `i_system.h` | `I_Error` for range-check error reporting (debug builds). |
| Core definitions | `doomdef.h` | `boolean`, `fixed_t`, `angle_t`, `FRACBITS`, `SCREENWIDTH`, and other fundamental types and constants. |
| Game state | `doomstat.h` | `extralight` (player light amplification), `skyflatnum` (sky flat index). |
| Renderer shared state | `r_local.h` | Aggregates all renderer sub-module headers; provides `curline`, `frontsector`, `backsector`, `viewz`, `viewangle`, `viewx`, `viewy`, `centeryfrac`, `xtoviewangle`, `floorclip`, `ceilingclip`, `ceilingplane`, `floorplane`, `scalelight`, `fixedcolormap`, `R_GetColumn`, `R_PointToDist`, `R_ScaleFromGlobalAngle`, `R_CheckPlane`, and column drawing globals (`dc_x`, `dc_yl`, `dc_yh`, `dc_texturemid`, `dc_colormap`, `dc_iscale`, `dc_source`, `colfunc`). |
| Sky rendering | `r_sky.h` | `skyflatnum` (also available via `doomstat.h` in some paths). |
| Fixed-point math | `m_fixed.h` (via `r_local.h`) | `FixedMul`, `FixedDiv`, `fixed_t`. |
| Lookup tables | `tables.h` (via `r_local.h`) | `finetangent`, `finesine`, `ANGLETOFINESHIFT`, and binary angle arithmetic. |
| Sprite/masked drawing | `r_things.c` (caller) | `R_DrawMaskedColumn`, `spryscale`, `sprtopscreen`, `mfloorclip`, `mceilingclip` - these globals are shared between `r_segs.c` and `r_things.c` through `r_local.h`. |

### Modules that depend on `r_segs.h`

| Module | How it uses `r_segs.h` |
|---|---|
| `r_local.h` | Includes `r_segs.h` as part of the renderer's aggregate internal header, making `R_RenderMaskedSegRange` available to all renderer sub-modules. |
| `r_things.c` | Calls `R_RenderMaskedSegRange` during `R_DrawMasked` to render deferred masked mid-textures interleaved with sprites. |
