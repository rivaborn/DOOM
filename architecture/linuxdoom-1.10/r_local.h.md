# File Overview

**File:** `linuxdoom-1.10/r_local.h`
**Module:** Renderer - Internal Aggregator Header

`r_local.h` is a convenience umbrella header that pulls together all of the internal renderer module headers into a single include. Any `.c` file that is part of the renderer subsystem includes `r_local.h` instead of listing each individual renderer header separately.

This pattern keeps individual source files clean while ensuring every renderer module has access to the full shared renderer interface: geometry types, lookup tables, rendering state, drawing functions, and utility routines.

---

## Purpose and Usage

When a renderer source file (e.g., `r_things.c`, `r_segs.c`, `r_bsp.c`, `r_plane.c`) writes:

```c
#include "r_local.h"
```

it receives, transitively, every declaration needed to participate in the rendering pipeline. The header itself defines nothing; it only includes other headers.

---

## Included Headers (in order)

### `tables.h`
Binary angle arithmetic and trigonometric lookup tables used throughout the renderer.

- `angle_t` type (unsigned 32-bit binary angle).
- `ANG45`, `ANG90`, `ANG180`, `ANG270`, `ANG360` constants.
- `finesine[]`, `finecosine[]` - 8192-entry trig tables indexed by `angle >> ANGLETOFINESHIFT`.
- `finetangent[]` - tangent table used for FOV and projection calculations.
- `ANGLETOFINESHIFT`, `FINEMASK` masking constants.

### `doomdef.h`
Global game definitions and screen dimensions.

- `SCREENWIDTH`, `SCREENHEIGHT` constants (320 x 200 for the original engine).
- `boolean`, `byte` type aliases.
- `fixed_t` (via `m_fixed.h`), `FRACUNIT`, `FRACBITS`.
- Game mode and skill level types.

### `r_data.h` (included twice - first occurrence)
Renderer data structures and WAD data access for textures and flats.

- Texture, flat, and sprite patch management (`R_InitData`, `R_FlatNumForName`, `R_TextureNumForName`, etc.).
- `spritewidth[]`, `spriteoffset[]`, `spritetopoffset[]` per-lump pixel arrays.
- `textureheight[]`, `texturecolumnlump[]`, `texturecolumnofs[]`, `texturetranslation[]`.
- `firstspritelump`, `lastspritelump`, `numspritelumps`.
- `firstflat`, `flattranslation[]`.

### `r_main.h`
Core renderer state, projection math, lighting tables, and function pointers.

- View parameters: `viewx`, `viewy`, `viewz`, `viewangle`, `viewcos`, `viewsin`.
- Viewport geometry: `viewwidth`, `viewheight`, `centerx`, `centery`, `centerxfrac`, `centeryfrac`, `projection`.
- Lighting tables: `scalelight[LIGHTLEVELS][MAXLIGHTSCALE]`, `zlight[LIGHTLEVELS][MAXLIGHTZ]`, `fixedcolormap`, `extralight`.
- Column/span function pointers: `colfunc`, `basecolfunc`, `fuzzcolfunc`, `spanfunc`.
- Utility functions: `R_PointOnSide`, `R_PointOnSegSide`, `R_PointToAngle`, `R_PointToDist`, `R_PointInSubsector`.
- `detailshift` (0 = high detail, 1 = low detail).
- `validcount` (frame counter for visited-sector deduplication).

### `r_bsp.h`
BSP traversal state.

- `drawsegs[]` array and `ds_p` pointer (the set of wall segments clipped and rendered this frame).
- `R_ClearClipSegs`, `R_ClearDrawSegs`, `R_RenderBSPNode`.
- `sscount` (visible subsector count, used for debugging).

### `r_segs.h`
Wall segment rendering.

- `R_RenderMaskedSegRange` - renders two-sided linedefs' masked mid-textures; called from `r_things.c` during sprite clipping.
- `rw_distance`, `rw_normalangle`, `rw_angle1` - shared segment rendering state.

### `r_plane.h`
Visplane (floor/ceiling) management and rendering.

- `visplane_t` pool functions: `R_InitPlanes`, `R_ClearPlanes`, `R_FindPlane`, `R_CheckPlane`, `R_DrawPlanes`.
- `floorplane`, `ceilingplane` current-sector pointers set during BSP traversal.
- `openings[]` / `lastopening` - shared clipping array for floor/ceiling span extents.

### `r_data.h` (second occurrence)
Repeated include (harmless due to include guard `__R_DATA__`). The comment "Include the refresh/render data structs." in the header indicates this was intentional for clarity, or the duplicate was not noticed.

### `r_things.h`
Sprite / thing rendering interface (see `r_things.h.md` for full details).

- `vissprites[]`, `vissprite_p`, `vsprsortedhead`.
- `mfloorclip`, `mceilingclip`, `spryscale`, `sprtopscreen`.
- `pspritescale`, `pspriteiscale`.
- `R_DrawMaskedColumn`, `R_SortVisSprites`, `R_AddSprites`, `R_InitSprites`, `R_ClearSprites`, `R_DrawMasked`.

### `r_draw.h`
Low-level column and span pixel writing (see `r_draw.h.md` for full details).

- Column renderer inputs: `dc_colormap`, `dc_x`, `dc_yl`, `dc_yh`, `dc_iscale`, `dc_texturemid`, `dc_source`.
- Span renderer inputs: `ds_y`, `ds_x1`, `ds_x2`, `ds_colormap`, `ds_xfrac`, `ds_yfrac`, `ds_xstep`, `ds_ystep`, `ds_source`.
- Column functions: `R_DrawColumn`, `R_DrawColumnLow`, `R_DrawFuzzColumn`, `R_DrawFuzzColumnLow`, `R_DrawTranslatedColumn`, `R_DrawTranslatedColumnLow`.
- Span functions: `R_DrawSpan`, `R_DrawSpanLow`.
- Buffer init: `R_InitBuffer`, `R_InitTranslationTables`.
- Border rendering: `R_FillBackScreen`, `R_DrawViewBorder`.
- `translationtables`, `dc_translation`.

---

## Data Structures

None defined here. All data structures used by the renderer are defined in `r_defs.h` (included transitively via `r_data.h`).

Key structures available after including `r_local.h`:
- `vissprite_t` - per-frame visible sprite record
- `drawseg_t` - per-frame rendered wall segment record
- `visplane_t` - per-frame visible floor/ceiling plane record
- `spritedef_t`, `spriteframe_t` - sprite animation/rotation data
- `column_t` (`post_t`) - masked patch column format
- `patch_t` - full patch/sprite image header
- `lighttable_t` - lighting palette type (`byte`)
- `vertex_t`, `seg_t`, `sector_t`, `subsector_t`, `node_t`, `line_t`, `side_t` - map geometry

---

## Dependencies

All headers listed above are direct dependencies. The full transitive dependency set also includes:

| File | Pulled in via |
|------|---------------|
| `m_fixed.h`   | `doomdef.h` or `r_defs.h` |
| `d_think.h`   | `r_defs.h` → `p_mobj.h` |
| `p_mobj.h`    | `r_defs.h` |
| `d_player.h`  | `r_main.h` |
| `info.h`      | `p_mobj.h` |

Include guard: `__R_LOCAL__`
