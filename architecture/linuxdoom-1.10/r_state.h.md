# File Overview

**File:** `linuxdoom-1.10/r_state.h`
**Module:** Renderer - Global Rendering State (Shared Externals)

`r_state.h` is a centralized declaration header for the global state variables that are shared between the renderer (`r_*` files) and the rest of the engine (particularly `p_setup.c`, `g_game.c`, `am_map.c`, and `st_stuff.c`). It is the authoritative list of renderer globals that non-renderer code is allowed to read.

The renderer modules themselves use `r_local.h` for their internal shared state. `r_state.h` is for code outside the renderer subsystem that needs read access to renderer-managed data without pulling in the entire internal renderer header tree.

---

## Global Variables

### Texture and sprite pixel data

| Type       | Name                | Purpose |
|------------|---------------------|---------|
| `fixed_t*` | `textureheight`     | Array of `numtextures` entries. Each entry is the height of the corresponding texture in fixed-point units (`actual_height * FRACUNIT`). Used for texture pegging (vertical alignment of upper/lower wall textures) in `r_segs.c`. |
| `fixed_t*` | `spritewidth`       | Array of `numspritelumps` entries. Width of each sprite patch in fixed-point (`actual_width * FRACUNIT`). Used by `R_ProjectSprite` to compute the sprite's right screen edge. |
| `fixed_t*` | `spriteoffset`      | Array of `numspritelumps` entries. Horizontal offset from the patch origin to its left edge in fixed-point. Used by `R_ProjectSprite` to correctly position the sprite's center relative to the thing's map position. |
| `fixed_t*` | `spritetopoffset`   | Array of `numspritelumps` entries. Vertical offset from the patch origin to its top edge in fixed-point. Used by `R_ProjectSprite` to compute `vis->gzt` (world-space top of the sprite). |

### Color maps

| Type            | Name        | Purpose |
|-----------------|-------------|---------|
| `lighttable_t*` | `colormaps` | Base of the COLORMAP lump, a 32 x 256-byte block of palette remapping tables. `colormaps[0]` is the brightest (full-light) map; `colormaps[31]` is pitch black. Lighting computations index into this array. Shared with `doomstat.c` / `g_game.c`. |

### Viewport dimensions

| Type  | Name              | Purpose |
|-------|-------------------|---------|
| `int` | `viewwidth`       | Current viewport width in pixels. Equals `SCREENWIDTH` when view fills the screen; smaller when the player has reduced the view size. |
| `int` | `scaledviewwidth` | `viewwidth` scaled up for the border area when in low-detail mode. In normal detail mode equals `viewwidth`. Used to determine whether border rendering is needed. |
| `int` | `viewheight`      | Current viewport height in pixels. |

### Flat data

| Type  | Name        | Purpose |
|-------|-------------|---------|
| `int` | `firstflat` | WAD lump index of the first flat (floor/ceiling texture). Flats are identified by lump number relative to this base. Used in `r_plane.c` for flat texture lookup and in `r_sky.c` (`skyflatnum` is relative to this). |

### Animation translation tables

| Type   | Name               | Purpose |
|--------|--------------------|---------|
| `int*` | `flattranslation`  | Array of `numflats` entries. Maps a flat number to the current animated flat lump number. Updated each game tic by `P_UpdateSpecials` so that animated floor textures (e.g., lava, water) cycle through their frames. The renderer always reads through this table rather than using flat numbers directly. |
| `int*` | `texturetranslation` | Array of `numtextures` entries. Same concept for wall textures. Animated wall textures (switches, moving platforms' textures) are remapped here. |

### Sprite lump range

| Type  | Name             | Purpose |
|-------|------------------|---------|
| `int` | `firstspritelump`| WAD lump index of the first sprite patch. All `spriteframe_t.lump[]` values are offsets relative to this. |
| `int` | `lastspritelump` | WAD lump index of the last sprite patch (inclusive). |
| `int` | `numspritelumps` | Total count of sprite patch lumps (`lastspritelump - firstspritelump + 1`). |

### Sprite definition tables

| Type           | Name         | Purpose |
|----------------|--------------|---------|
| `int`          | `numsprites` | Total number of sprite names registered (length of `sprites[]`). |
| `spritedef_t*` | `sprites`    | Array of sprite definitions, one per sprite name (TROO, POSS, PLAY, etc.). Each entry holds the count of animation frames and a pointer to the `spriteframe_t` array for that sprite. Built by `R_InitSpriteDefs` in `r_things.c`. |

### Map geometry arrays

These are allocated by `P_LoadXxx` functions in `p_setup.c` and filled from the WAD's map lumps. The renderer accesses them read-only during traversal and rendering.

| Type           | Name           | Count variable   | Purpose |
|----------------|----------------|------------------|---------|
| `vertex_t*`    | `vertexes`     | `numvertexes`    | Map vertex coordinates. |
| `seg_t*`       | `segs`         | `numsegs`        | Line segments (half-edges of linedefs). Each seg references its sidedef and is used by `r_segs.c` to render wall columns. |
| `sector_t*`    | `sectors`      | `numsectors`     | Sector records (floor/ceiling heights, light level, texture IDs, thing lists). |
| `subsector_t*` | `subsectors`   | `numsubsectors`  | BSP leaf nodes; each references a sector and a range of segs. The BSP traversal visits these to identify visible areas. |
| `node_t*`      | `nodes`        | `numnodes`       | BSP internal nodes with partition lines and child bounding boxes. |
| `line_t*`      | `lines`        | `numlines`       | Linedef records (wall boundaries, specials). |
| `side_t*`      | `sides`        | `numsides`       | Sidedef records (texture IDs, offsets for a single face of a linedef). |

### Point-of-view state

| Type        | Name          | Purpose |
|-------------|---------------|---------|
| `fixed_t`   | `viewx`       | Player camera X coordinate in world space (fixed-point). |
| `fixed_t`   | `viewy`       | Player camera Y coordinate in world space (fixed-point). |
| `fixed_t`   | `viewz`       | Player camera Z (eye height) in world space (fixed-point). |
| `angle_t`   | `viewangle`   | Camera horizontal facing direction as a binary angle. |
| `player_t*` | `viewplayer`  | Pointer to the player whose view is being rendered. Used for weapon sprite rendering and lighting lookups. |

### Angle and projection lookup tables

| Type       | Name                          | Purpose |
|------------|-------------------------------|---------|
| `angle_t`  | `clipangle`                   | Half-FOV angle. The visible horizontal range spans `[viewangle - clipangle, viewangle + clipangle]`. |
| `int`      | `viewangletox[FINEANGLES/2]`  | Maps a fine angle (half-circle) to the screen X column that ray would hit. Used in `r_bsp.c` for frustum culling. Size is `FINEANGLES/2 = 2048` entries. |
| `angle_t`  | `xtoviewangle[SCREENWIDTH+1]` | Inverse of `viewangletox`: maps a screen X column to the view-relative angle of that column. Used in `r_segs.c`, `r_plane.c`, and `r_sky.c`. `SCREENWIDTH+1` entries (320+1=321) to include both edges. |

### Segment rendering shared state

| Type      | Name            | Purpose |
|-----------|-----------------|---------|
| `fixed_t` | `rw_distance`   | Perpendicular distance from the viewpoint to the current wall segment, computed in `r_bsp.c` and used by `r_segs.c`. |
| `angle_t` | `rw_normalangle`| Angle perpendicular to the current wall segment. Used with `rw_distance` to compute per-column scale in `r_segs.c`. |
| `int`     | `rw_angle1`     | Angle from the viewpoint to the left end of the current wall segment. Used in `R_ScaleFromGlobalAngle`. |
| `int`     | `sscount`       | Count of visible subsectors rendered this frame (debug/stats counter). |

### Current-sector visplanes

| Type          | Name           | Purpose |
|---------------|----------------|---------|
| `visplane_t*` | `floorplane`   | Visplane being accumulated for the current subsector's floor. Set by `R_Subsector` in `r_bsp.c` via `R_FindPlane`; extended column-by-column as wall segs are rendered. |
| `visplane_t*` | `ceilingplane` | Visplane being accumulated for the current subsector's ceiling. |

---

## Data Structures

None defined in this header. All types referenced (`fixed_t`, `spritedef_t`, `vertex_t`, `seg_t`, `sector_t`, `subsector_t`, `node_t`, `line_t`, `side_t`, `visplane_t`, `angle_t`, `player_t`, `lighttable_t`) are defined in their respective headers.

---

## Dependencies

| File        | Reason |
|-------------|--------|
| `d_player.h`| Provides `player_t` type for `viewplayer`; also pulls in `p_mobj.h`, `d_think.h` |
| `r_data.h`  | Provides `spritedef_t`, `spriteframe_t`, and the renderer data type definitions; also pulls in `r_defs.h` for `visplane_t`, `drawseg_t`, `vertex_t`, `sector_t`, etc. |

Include guard: `__R_STATE__`
