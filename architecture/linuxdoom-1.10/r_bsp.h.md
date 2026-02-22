# File Overview

`r_bsp.h` is the public interface header for the BSP traversal and wall-segment clipping subsystem implemented in `r_bsp.c`. It exposes the external variables and functions that other rendering modules (particularly `r_segs.c`, `r_things.c`, and `r_main.c`) need to interact with the BSP rendering state.

This header is included via `r_local.h`, which aggregates all renderer sub-module headers.

## Global Variables (extern declarations)

| Type | Name | Description |
|---|---|---|
| `seg_t*` | `curline` | The wall segment currently being rendered. Set by `R_AddLine` and used by `R_StoreWallRange` in `r_segs.c`. |
| `side_t*` | `sidedef` | The sidedef of the current segment, set during wall storage. |
| `line_t*` | `linedef` | The linedef of the current segment, set during wall storage. |
| `sector_t*` | `frontsector` | The sector on the viewer's side of the current segment. |
| `sector_t*` | `backsector` | The sector behind the current segment (NULL for one-sided walls). |
| `int` | `rw_x` | Current screen X column being rendered within a wall range, updated by `R_RenderSegLoop` in `r_segs.c`. |
| `int` | `rw_stopx` | One past the last screen X column of the current wall range. |
| `boolean` | `segtextured` | True if the current segment has any visible textures (mid, top, or bottom). Used to avoid texture-column calculations for untextured geometry. |
| `boolean` | `markfloor` | True if the floor visplane needs to be marked for the current segment's column range. |
| `boolean` | `markceiling` | True if the ceiling visplane needs to be marked for the current segment's column range. |
| `boolean` | `skymap` | (Referenced but not prominently used in this version.) Indicates sky rendering context. |
| `drawseg_t` | `drawsegs[MAXDRAWSEGS]` | Array of draw-segment records for the current frame, used by sprite rendering in `r_things.c` for occlusion determination. `MAXDRAWSEGS` is 256. |
| `drawseg_t*` | `ds_p` | Pointer to the next free draw-segment slot. |
| `lighttable_t**` | `hscalelight` | Horizontal-scale light table (unused in final code but declared). |
| `lighttable_t**` | `vscalelight` | Vertical-scale light table (unused in final code but declared). |
| `lighttable_t**` | `dscalelight` | Distance-scale light table (unused in final code but declared). |

## Data Structures

### `drawfunc_t` (typedef)

```c
typedef void (*drawfunc_t)(int start, int stop);
```

A function pointer type for column/span drawing functions taking a start and stop coordinate. Used to abstract the drawing of wall columns at different detail levels. In practice, `colfunc`, `spanfunc`, etc. in `r_main.c` serve this role; `drawfunc_t` appears in the header as a type alias but is not heavily used in the final codebase.

## Functions

### `R_ClearClipSegs`

```c
void R_ClearClipSegs(void);
```

Resets the solid-clip segment list to its initial (empty/sentinel-only) state at the start of each frame. Called from `R_RenderPlayerView` in `r_main.c`.

### `R_ClearDrawSegs`

```c
void R_ClearDrawSegs(void);
```

Resets the draw-segment pointer `ds_p` to the beginning of `drawsegs`, discarding previous frame's records. Called from `R_RenderPlayerView` in `r_main.c`.

### `R_RenderBSPNode`

```c
void R_RenderBSPNode(int bspnum);
```

**Purpose:** Entry point for BSP tree traversal. Recursively renders all potentially visible subsectors in front-to-back order, starting from the root node.

**Parameters:**
- `bspnum` (`int`): The index of the starting BSP node. Pass `numnodes - 1` to start at the root (as done in `R_RenderPlayerView`).

**Return Value:** `void`

See `r_bsp.c` for full implementation details.

## Dependencies

| File | Reason |
|---|---|
| `r_defs.h` | Definitions of `seg_t`, `side_t`, `line_t`, `sector_t`, `drawseg_t`, `MAXDRAWSEGS`, `lighttable_t` |
| Implicitly `doomdef.h` | For `boolean`, `fixed_t`, `angle_t` via the types used in variable declarations |
