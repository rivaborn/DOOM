# File Overview

**Path:** `linuxdoom-1.10/r_main.c`
**Module:** Renderer - Main Entry Point and Setup

`r_main.c` is the central hub of DOOM's software renderer. It owns the renderer initialization sequence, the per-frame viewpoint setup, and the top-level rendering pipeline. Every rendered frame passes through `R_RenderPlayerView`, which this file defines. It also provides the fundamental geometric utilities - BSP point-on-side tests, angle/distance calculations - that the rest of the renderer relies on, along with the two lookup tables (`viewangletox[]` and `xtoviewangle[]`) that map between screen X coordinates and view angles.

### The Rendering Pipeline

DOOM renders a frame in four sequential phases, all driven by `R_RenderPlayerView`:

```
R_RenderPlayerView(player)
  |
  +-- R_SetupFrame()          -- capture viewpoint, color maps, counters
  |
  +-- R_ClearClipSegs()       -- reset solid-seg occlusion list
  +-- R_ClearDrawSegs()       -- reset draw-seg list
  +-- R_ClearPlanes()         -- reset visplane list and clip arrays
  +-- R_ClearSprites()        -- reset vissprite list
  |
  +-- R_RenderBSPNode()       -- front-to-back BSP traversal
  |     calls R_StoreWallRange() for each visible seg -> fills draw-seg list
  |     marks floor/ceiling visplane entries per column
  |
  +-- R_DrawPlanes()          -- rasterise every accumulated visplane
  |     floors and ceilings as horizontal spans (spanfunc)
  |     sky columns via colfunc
  |
  +-- R_DrawMasked()          -- sprites and masked mid-textures (back-to-front)
```

### The Column/Span Duality

DOOM renders walls as vertical **columns** (one per screen X), and floors/ceilings as horizontal **spans** (one per screen Y within a visplane). Function pointers (`colfunc`, `spanfunc`, etc.) allow the detail level to swap these routines at runtime without branching inside the inner loops.

### Fixed-Point Projection

The projection model is a flat screen at distance `projection = centerxfrac` (half the view width in fixed-point). Scale at a given wall is:

```
scale = (projection * sin(angleb)) / (rw_distance * sin(anglea))
```

where `anglea` is the angle from the view direction to the column and `angleb` is the angle from the wall normal to the column.

---

## Global Variables

| Type | Name | Purpose |
|------|------|---------|
| `int` | `viewangleoffset` | Added to the player's map angle; used for network player view offsets. |
| `int` | `validcount` | Monotonically incrementing frame stamp. BSP and other traversal code stamps structures with this value to avoid revisiting them in the same frame. Incremented each frame by `R_SetupFrame`. |
| `lighttable_t*` | `fixedcolormap` | Non-NULL when the player has a powerup (invulnerability or light-amp goggles) that forces a single colormap for all rendering, bypassing the normal distance/scale lighting. |
| `int` | `centerx` | Horizontal center of the view window in pixels. Equal to `viewwidth / 2`. |
| `int` | `centery` | Vertical center of the view window in pixels. Equal to `viewheight / 2`. |
| `fixed_t` | `centerxfrac` | `centerx` in 16.16 fixed-point. Also used as the projection distance. |
| `fixed_t` | `centeryfrac` | `centery` in 16.16 fixed-point. |
| `fixed_t` | `projection` | Perspective projection constant. Equals `centerxfrac`. Used in `R_ScaleFromGlobalAngle` and plane slope calculation. |
| `int` | `framecount` | Total frames rendered since startup; used for profiling. |
| `int` | `sscount` | Subsectors rendered this frame; profiling only. |
| `int` | `linecount` | Segs processed this frame; profiling only. |
| `int` | `loopcount` | Inner loop iterations this frame; profiling only. |
| `fixed_t` | `viewx` | Player viewpoint X in map coordinates (16.16 fixed-point). |
| `fixed_t` | `viewy` | Player viewpoint Y in map coordinates (16.16 fixed-point). |
| `fixed_t` | `viewz` | Player eye height in map coordinates (16.16 fixed-point). |
| `angle_t` | `viewangle` | Player viewing direction as a DOOM binary angle (0 = east, ANG90 = north). |
| `fixed_t` | `viewcos` | Cosine of `viewangle` in fixed-point, precomputed each frame. |
| `fixed_t` | `viewsin` | Sine of `viewangle` in fixed-point, precomputed each frame. |
| `player_t*` | `viewplayer` | Pointer to the player being rendered. |
| `int` | `detailshift` | 0 = high detail (full resolution), 1 = low detail (half horizontal resolution). Controls which `colfunc`/`spanfunc` variants are active. |
| `angle_t` | `clipangle` | Half the horizontal field of view, expressed as a binary angle. Equals `xtoviewangle[0]`. Segs outside `[-clipangle, +clipangle]` relative to `viewangle` are culled. |
| `int` | `viewangletox[FINEANGLES/2]` | Lookup table: fine angle index -> screen X column. Many angles map to the same X. Values are clamped to `[-1, viewwidth+1]`. Size: 2048 entries. |
| `angle_t` | `xtoviewangle[SCREENWIDTH+1]` | Lookup table: screen X column -> the leftmost (smallest magnitude) view angle that maps to that column. Inverse of `viewangletox`. |
| `fixed_t*` | `finecosine` | Pointer into `finesine[]` offset by `FINEANGLES/4`. Gives cosine via the sine table: `cos(a) = sin(a + 90deg)`. |
| `lighttable_t*` | `scalelight[LIGHTLEVELS][MAXLIGHTSCALE]` | 2D LUT indexed by `[light_level][scale]`. Maps a wall's scale (proxy for distance) to the appropriate colormap. Built in `R_ExecuteSetViewSize` because it depends on view width. |
| `lighttable_t*` | `scalelightfixed[MAXLIGHTSCALE]` | When `fixedcolormap` is active, all entries in this array point to that same colormap. `walllights` is redirected here so the wall loop needs no special case. |
| `lighttable_t*` | `zlight[LIGHTLEVELS][MAXLIGHTZ]` | 2D LUT indexed by `[light_level][z_distance_bucket]`. Used for floor/ceiling span lighting. Built once in `R_InitLightTables` (independent of view size). |
| `int` | `extralight` | Additional light level from the player's gun flash or powerup. Added to the sector's `lightlevel` before indexing `scalelight`/`zlight`. |
| `void (*colfunc)(void)` | `colfunc` | Current column-drawing function. Points to `R_DrawColumn` (high) or `R_DrawColumnLow` (low detail). |
| `void (*basecolfunc)(void)` | `basecolfunc` | The non-special column function (normal, non-fuzz, non-translated). Used to restore `colfunc` after drawing fuzz columns. |
| `void (*fuzzcolfunc)(void)` | `fuzzcolfunc` | Column function for the Spectre/Invisible enemy fuzz effect. Always `R_DrawFuzzColumn`. |
| `void (*transcolfunc)(void)` | `transcolfunc` | Column function for color-translated sprites (player skins in multiplayer). |
| `void (*spanfunc)(void)` | `spanfunc` | Current span-drawing function. Points to `R_DrawSpan` (high) or `R_DrawSpanLow` (low detail). |
| `boolean` | `setsizeneeded` | Flag set by `R_SetViewSize`; tells `R_ExecuteSetViewSize` that a recalculation is pending. |
| `int` | `setblocks` | Requested view-window block count (1-11) passed to the deferred `R_ExecuteSetViewSize`. |
| `int` | `setdetail` | Requested detail level (0=high, 1=low) for the deferred `R_ExecuteSetViewSize`. |

---

## Functions

### `R_AddPointToBox`
```c
void R_AddPointToBox(int x, int y, fixed_t* box)
```
**Purpose:** Expands a 4-element bounding box array (`BOXLEFT`, `BOXRIGHT`, `BOXBOTTOM`, `BOXTOP`) to include the point `(x, y)`. Used during BSP bounding-box construction.

**Parameters:**
- `x`, `y` - Map coordinates of the point to add.
- `box` - Pointer to a 4-element `fixed_t` bounding box array.

**Return:** None.

---

### `R_PointOnSide`
```c
int R_PointOnSide(fixed_t x, fixed_t y, node_t* node)
```
**Purpose:** Determines which side (front = 0, back = 1) of a BSP partition line the point `(x, y)` lies on. Called by `R_RenderBSPNode` at every BSP node during traversal and by `R_PointInSubsector`.

**Key logic:** Uses sign-bit tricks to avoid a full cross-product multiply when the signs of the operands already determine the result. Falls back to `FixedMul(node->dy, dx)` vs `FixedMul(dy, node->dx)` cross-product comparison. Special cases for axis-aligned partition lines (when `node->dx == 0` or `node->dy == 0`).

**Parameters:**
- `x`, `y` - Map coordinates to test.
- `node` - BSP partition node.

**Return:** `0` (front/right side) or `1` (back/left side).

---

### `R_PointOnSegSide`
```c
int R_PointOnSegSide(fixed_t x, fixed_t y, seg_t* line)
```
**Purpose:** Same cross-product side test as `R_PointOnSide` but operates on a `seg_t` rather than a BSP `node_t`. Used to determine which side of a map line segment a point is on (e.g. for sprite ordering).

**Parameters:**
- `x`, `y` - Map coordinates to test.
- `line` - Seg whose endpoints `v1`, `v2` define the partition.

**Return:** `0` (front) or `1` (back).

---

### `R_PointToAngle`
```c
angle_t R_PointToAngle(fixed_t x, fixed_t y)
```
**Purpose:** Converts a world-space point `(x, y)` to the view angle (relative to `viewx`, `viewy`) needed to see it. This is the fundamental angle-from-camera function used throughout the renderer.

**Key logic:** Subtracts the viewpoint, then classifies the delta into one of eight octants. Within each octant the smaller coordinate is divided by the larger to produce a slope in `[0, 1]`, which is looked up in `tantoangle[]` to get the fine angle, then the octant offset is added.

**Parameters:**
- `x`, `y` - World-space map coordinates of the target point.

**Return:** `angle_t` binary angle from the viewer to `(x, y)`.

---

### `R_PointToAngle2`
```c
angle_t R_PointToAngle2(fixed_t x1, fixed_t y1, fixed_t x2, fixed_t y2)
```
**Purpose:** Computes the angle from an arbitrary origin `(x1, y1)` to a target `(x2, y2)`. Temporarily overrides `viewx`/`viewy` to use `R_PointToAngle`.

**Note:** This function has a global side-effect: it overwrites `viewx` and `viewy` with `x1` and `y1`. Callers must be aware that these globals will be modified.

**Return:** `angle_t` binary angle from `(x1, y1)` to `(x2, y2)`.

---

### `R_PointToDist`
```c
fixed_t R_PointToDist(fixed_t x, fixed_t y)
```
**Purpose:** Computes the Euclidean distance from the current viewpoint to `(x, y)`. Used when calculating the perpendicular wall distance for scale computation.

**Key logic:** Swaps `dx`/`dy` so the larger is in `dx`. Computes the slope `dy/dx`, looks up `atan` in `tantoangle`, adds 90 degrees to get the complementary angle, then uses the sine of that angle (which equals the cosine of the original) to recover distance as `dx / cos(angle)` = `dx / finesine[angle+90]`.

**Return:** `fixed_t` distance in map units (16.16 fixed-point).

---

### `R_InitPointToAngle`
```c
void R_InitPointToAngle(void)
```
**Purpose:** Stub function. Originally populated the `tantoangle[]` lookup table; now the table is pre-baked in `tables.c` and this function is a no-op.

---

### `R_ScaleFromGlobalAngle`
```c
fixed_t R_ScaleFromGlobalAngle(angle_t visangle)
```
**Purpose:** Returns the perspective scale factor for a wall column at the given view angle. The scale is the ratio of screen pixels to world units at that column, driving both texture mapping (`dc_iscale = 1/scale`) and the height of the column on screen.

**Key formula:**
```
anglea = ANG90 + (visangle - viewangle)
angleb = ANG90 + (visangle - rw_normalangle)
num = projection * sin(angleb) * (1 << detailshift)
den = rw_distance * sin(anglea)
scale = num / den
```
Scale is clamped to `[256, 64*FRACUNIT]` to prevent division instabilities at extreme angles.

**Preconditions:** `rw_distance` and `rw_normalangle` must already be set (done in `R_StoreWallRange` before this is called).

**Return:** `fixed_t` scale factor (16.16 fixed-point). Larger = closer wall = taller column.

---

### `R_InitTables`
```c
void R_InitTables(void)
```
**Purpose:** Stub function. Originally computed `finetangent[]` and `finesine[]`; now these tables come from `tables.c`. The function body is `#if 0`'d out.

---

### `R_InitTextureMapping`
```c
void R_InitTextureMapping(void)
```
**Purpose:** Builds the two angle-to-X and X-to-angle lookup tables used by the entire renderer.

**Key logic:**
1. Computes `focallength = centerxfrac / tan(FIELDOFVIEW/2)`. This is the distance from the eye to the projection plane such that the full `FIELDOFVIEW` (2048 fine-angle units = 90 degrees) maps to `viewwidth` pixels.
2. For each of the 2048 fine-angle entries, evaluates `finetangent[i] * focallength` to get the screen X and stores it in `viewangletox[i]`.
3. Inverts the mapping: scans `viewangletox` for each screen column `x` to find the first angle that maps to `x`, storing it in `xtoviewangle[x]`.
4. Sets `clipangle = xtoviewangle[0]` (the outermost view angle).

**Called by:** `R_ExecuteSetViewSize` whenever the view size changes.

---

### `R_InitLightTables`
```c
void R_InitLightTables(void)
```
**Purpose:** Fills the `zlight[LIGHTLEVELS][MAXLIGHTZ]` table that maps a sector light level and a Z-distance bucket to a colormap. This table is used when rendering floor and ceiling spans.

**Key formula:** For sector light level `i` and Z-bucket `j`:
```
startmap = ((LIGHTLEVELS-1-i)*2) * NUMCOLORMAPS / LIGHTLEVELS
scale = (SCREENWIDTH/2 * FRACUNIT) / ((j+1) << LIGHTZSHIFT)
level = startmap - scale / DISTMAP
```
Lower `j` (closer) or higher light level produces a lower colormap index (brighter). The result is clamped to `[0, NUMCOLORMAPS-1]`.

**Called by:** `R_Init` once at startup. Independent of view size.

---

### `R_SetViewSize`
```c
void R_SetViewSize(int blocks, int detail)
```
**Purpose:** Deferred view-size change request. Sets flags and stores the requested `blocks` (view width in 32-pixel units, 1-11) and `detail` level for application on the next frame boundary. Avoids recalculating tables mid-frame.

**Parameters:**
- `blocks` - View window size: 1-10 = windowed (`blocks*32` wide), 11 = full screen.
- `detail` - `0` = high detail, `1` = low detail.

---

### `R_ExecuteSetViewSize`
```c
void R_ExecuteSetViewSize(void)
```
**Purpose:** Actually applies a queued view-size change. This is one of the most important setup functions - it recalculates every per-view-size constant and table.

**What it does:**
1. Computes `scaledviewwidth`, `viewheight`, `viewwidth` from `setblocks`/`setdetail`.
2. Recomputes `centerx`, `centery`, `centerxfrac`, `centeryfrac`, `projection`.
3. Selects the appropriate column/span function pointers based on `detailshift`.
4. Calls `R_InitBuffer` to set up the video buffer offset table.
5. Calls `R_InitTextureMapping` to rebuild `viewangletox`/`xtoviewangle`.
6. Recomputes `pspritescale`/`pspriteiscale` for player weapon sprites.
7. Fills `screenheightarray[x] = viewheight` (used for sprite top-clipping on solid walls).
8. Fills `yslope[y]` for floor/ceiling distance calculation: `yslope[y] = (viewwidth/2 * FRACUNIT) / |y - viewheight/2|`. Larger at the horizon, smaller near the edges.
9. Fills `distscale[x]` for floor/ceiling angle correction: `1 / cos(xtoviewangle[x])`. Corrects for the perspective angle when computing floor distances at oblique columns.
10. Rebuilds the `scalelight[LIGHTLEVELS][MAXLIGHTSCALE]` table (depends on `viewwidth`).

---

### `R_Init`
```c
void R_Init(void)
```
**Purpose:** Top-level renderer initialization called once at game startup. Calls every renderer sub-system initializer in order.

**Initialization sequence:**
1. `R_InitData()` - textures, flats, sprites from WAD.
2. `R_InitPointToAngle()` - stub.
3. `R_InitTables()` - stub.
4. `R_SetViewSize(screenblocks, detailLevel)` - queue the initial view size.
5. `R_InitPlanes()` - stub.
6. `R_InitLightTables()` - build `zlight[][]`.
7. `R_InitSkyMap()` - precompute sky texture mid offset.
8. `R_InitTranslationTables()` - build colormap translation tables for player colors.
9. Zeros `framecount`.

---

### `R_PointInSubsector`
```c
subsector_t* R_PointInSubsector(fixed_t x, fixed_t y)
```
**Purpose:** Traverses the BSP tree from the root to find the subsector (convex BSP leaf) that contains the world-space point `(x, y)`. Used to determine which sector a point is in.

**Key logic:** Starts at `numnodes-1` (the BSP root). At each node calls `R_PointOnSide` and follows the appropriate child until the `NF_SUBSECTOR` flag is set in the child index, indicating a leaf. Handles the degenerate single-subsector case (no BSP tree).

**Return:** Pointer to the `subsector_t` containing the point.

---

### `R_SetupFrame`
```c
void R_SetupFrame(player_t* player)
```
**Purpose:** Per-frame initialization of all viewpoint globals. Called at the start of `R_RenderPlayerView`.

**What it sets:**
- `viewplayer`, `viewx`, `viewy`, `viewz` from the player's map object.
- `viewangle = player->mo->angle + viewangleoffset`.
- `extralight` from the player's gun-flash counter.
- `viewsin`, `viewcos` by indexing `finesine`/`finecosine` with the fine angle.
- `fixedcolormap` and `walllights` redirect if the player has a fixed colormap powerup (invulnerability/goggles). If so, `scalelightfixed` is filled with the fixed colormap and `walllights` points there.
- Increments `framecount` and `validcount`.

---

### `R_RenderPlayerView`
```c
void R_RenderPlayerView(player_t* player)
```
**Purpose:** The top-level render entry point. Called once per game tick by the game-loop drawing code. Orchestrates the complete rendering of one frame from the player's viewpoint.

**Pipeline:**
1. `R_SetupFrame(player)` - set viewpoint globals.
2. `R_ClearClipSegs()` - reset the solid-seg occlusion clipper.
3. `R_ClearDrawSegs()` - reset the draw-seg list (`ds_p = drawsegs`).
4. `R_ClearPlanes()` - reset visplane list; set `floorclip[x]=viewheight`, `ceilingclip[x]=-1`; compute `basexscale`/`baseyscale`.
5. `R_ClearSprites()` - reset the vissprite list.
6. `NetUpdate()` - check for incoming network packets (sprinkled throughout to keep latency low).
7. `R_RenderBSPNode(numnodes-1)` - front-to-back BSP traversal. This is where all wall segs are processed and draw-segs and visplane spans are accumulated.
8. `NetUpdate()`.
9. `R_DrawPlanes()` - rasterise all accumulated floor/ceiling visplanes.
10. `NetUpdate()`.
11. `R_DrawMasked()` - draw sprites and masked mid-textures back-to-front.
12. `NetUpdate()`.

---

## Data Structures

No new structs are defined in this file. It references `node_t`, `seg_t`, `subsector_t`, and `player_t` from other headers.

### Lighting Constants (defined in `r_main.h`)

| Constant | Value | Meaning |
|----------|-------|---------|
| `LIGHTLEVELS` | 16 | Number of distinct sector light levels used for rendering. |
| `LIGHTSEGSHIFT` | 4 | Right-shift to convert a `sector->lightlevel` (0-255) to a light-level index (0-15). |
| `MAXLIGHTSCALE` | 48 | Maximum scale index for the `scalelight` table (wall distance-based lighting). |
| `LIGHTSCALESHIFT` | 12 | Right-shift applied to a wall's scale to get the `scalelight` column index. |
| `MAXLIGHTZ` | 128 | Maximum Z-distance index for the `zlight` table (floor/ceiling lighting). |
| `LIGHTZSHIFT` | 20 | Right-shift applied to a floor/ceiling distance to get the `zlight` column index. |
| `NUMCOLORMAPS` | 32 | Total number of colormaps in the `COLORMAP` lump (brightest to darkest). |
| `FIELDOFVIEW` | 2048 | Horizontal field of view in fine-angle units. 2048 = 90 degrees (FINEANGLES/4). |
| `DISTMAP` | 2 | Divisor used when building `zlight` to control the rate at which lighting falls off with distance. |

---

## Dependencies

| Module | Usage |
|--------|-------|
| `r_local.h` | Master renderer header; pulls in `r_main.h`, `r_bsp.h`, `r_segs.h`, `r_plane.h`, `r_draw.h`, `r_things.h`, `r_data.h`, `tables.h`. |
| `r_sky.h` | `R_InitSkyMap()` called during `R_Init`. |
| `doomdef.h` | Screen dimensions, `FRACUNIT`, `FRACBITS`, `boolean`, `fixed_t`. |
| `d_net.h` | `NetUpdate()` called inside `R_RenderPlayerView` to service network. |
| `m_bbox.h` | `BOXLEFT`, `BOXRIGHT`, `BOXBOTTOM`, `BOXTOP` constants for `R_AddPointToBox`. |
| `tables.h` | `finesine[]`, `tantoangle[]`, `SlopeDiv()`, `ANG90`, `ANG180`, etc. |
| `m_fixed.h` | `FixedMul`, `FixedDiv`, `fixed_t`. |
| `r_draw.c` | Column functions `R_DrawColumn`, `R_DrawColumnLow`, `R_DrawFuzzColumn`, `R_DrawTranslatedColumn`, `R_DrawSpan`, `R_DrawSpanLow`; buffer init `R_InitBuffer`. |
| `r_data.c` | `R_InitData()`, `R_InitTranslationTables()`, `R_GetColumn()`, `colormaps`, `texturetranslation[]`, `textureheight[]`. |
| `r_plane.c` | `R_InitPlanes()`, `R_ClearPlanes()`, `R_DrawPlanes()`, `yslope[]`, `distscale[]`. |
| `r_bsp.c` | `R_RenderBSPNode()`, `R_ClearClipSegs()`, `R_ClearDrawSegs()`. |
| `r_things.c` | `R_ClearSprites()`, `R_DrawMasked()`, `pspritescale`, `pspriteiscale`, `screenheightarray[]`, `negonearray[]`. |
| `doomstat.h` | Game-state globals such as `numnodes`, `nodes`, `subsectors`. |
