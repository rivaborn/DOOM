# File Overview

**Path:** `linuxdoom-1.10/r_main.h`
**Module:** Renderer - Main Public Interface Header

`r_main.h` is the public interface for the renderer's core module. It declares every symbol that other subsystems need to call into the renderer or inspect its per-frame state. It groups declarations into three logical areas:

1. **Point-of-view (POV) variables** - the viewpoint position, screen geometry, and projection constants that any renderer module may read.
2. **Lighting tables and constants** - the colormap LUT arrays and the constants that define how many light levels and scale buckets exist.
3. **Function pointer declarations** - the `colfunc`/`spanfunc` indirection layer that allows high/low-detail and special-effects rendering to be selected at runtime.
4. **Public function prototypes** - geometry utilities and the three externally visible control functions (`R_Init`, `R_SetViewSize`, `R_RenderPlayerView`).

This header is included (indirectly via `r_local.h`) by every renderer source file. External game code (`g_game.c`, `m_menu.c`) includes it directly to call `R_RenderPlayerView`, `R_Init`, and `R_SetViewSize`.

---

## Global Variables

All variables below are declared `extern` here and defined in `r_main.c`.

### Point-of-View Variables

| Type | Name | Purpose |
|------|------|---------|
| `fixed_t` | `viewcos` | Cosine of the current view angle (16.16 fixed-point). Updated each frame by `R_SetupFrame`. |
| `fixed_t` | `viewsin` | Sine of the current view angle (16.16 fixed-point). Updated each frame by `R_SetupFrame`. |
| `int` | `viewwidth` | Width of the rendered view window in pixels. Equals `scaledviewwidth >> detailshift`. |
| `int` | `viewheight` | Height of the rendered view window in pixels. |
| `int` | `viewwindowx` | X pixel offset of the view window within the screen framebuffer. |
| `int` | `viewwindowy` | Y pixel offset of the view window within the screen framebuffer. |
| `int` | `centerx` | Horizontal center pixel of the view window. |
| `int` | `centery` | Vertical center pixel of the view window. |
| `fixed_t` | `centerxfrac` | `centerx` in 16.16 fixed-point. Doubles as the projection focal length. |
| `fixed_t` | `centeryfrac` | `centery` in 16.16 fixed-point. Used in wall and plane vertical clipping. |
| `fixed_t` | `projection` | Projection constant = `centerxfrac`. The distance from the eye to the projection plane in the flat-projection model. |
| `int` | `validcount` | Frame stamp counter. Incremented every frame to allow O(1) "already visited this frame" checks throughout the renderer and BSP code. |
| `int` | `linecount` | Profiling counter: seg edges processed this frame. |
| `int` | `loopcount` | Profiling counter: inner loop iterations this frame. |

### Lighting Variables and Constants

| Type | Name | Purpose |
|------|------|---------|
| `lighttable_t*` | `scalelight[LIGHTLEVELS][MAXLIGHTSCALE]` | 2D colormap LUT for wall columns, indexed by `[sector_lightlevel >> LIGHTSEGSHIFT][column_scale >> LIGHTSCALESHIFT]`. Closer (larger scale) walls are brighter. |
| `lighttable_t*` | `scalelightfixed[MAXLIGHTSCALE]` | Fixed-colormap override. When a player has invulnerability/goggles, all entries point to the same map. `walllights` is redirected here. |
| `lighttable_t*` | `zlight[LIGHTLEVELS][MAXLIGHTZ]` | 2D colormap LUT for floor/ceiling spans, indexed by `[sector_lightlevel >> LIGHTSEGSHIFT][z_distance >> LIGHTZSHIFT]`. |
| `int` | `extralight` | Bonus light from a gun flash or powerup. Added before indexing `scalelight`/`zlight`. |
| `lighttable_t*` | `fixedcolormap` | When non-NULL (powerup active), every renderer module uses this colormap exclusively instead of distance-based dimming. |
| `int` | `detailshift` | 0 = high detail, 1 = low detail. Controls horizontal pixel doubling and which draw routines are active. |

#### Lighting Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `LIGHTLEVELS` | 16 | Number of distinct brightness bands. Sector `lightlevel` (0-255) is right-shifted by `LIGHTSEGSHIFT` (4) to produce an index 0-15. |
| `LIGHTSEGSHIFT` | 4 | Shift to convert `sector->lightlevel` to a light-level index. |
| `MAXLIGHTSCALE` | 48 | Maximum column index for `scalelight`. A wall scale >= 48 (after shifting) is clamped to 47. |
| `LIGHTSCALESHIFT` | 12 | Right-shift to convert wall `scale` to a `scalelight` column index. |
| `MAXLIGHTZ` | 128 | Maximum Z-distance bucket index for `zlight`. |
| `LIGHTZSHIFT` | 20 | Right-shift to convert a distance value to a `zlight` index. |
| `NUMCOLORMAPS` | 32 | Total colormaps in the WAD's COLORMAP lump. Index 0 = fullbright, 31 = near-black. |

### Function Pointer Declarations

These allow runtime selection of rendering routines. Set by `R_ExecuteSetViewSize`.

| Type | Name | Purpose |
|------|------|---------|
| `void (*colfunc)(void)` | `colfunc` | Current wall/sprite column renderer. `R_DrawColumn` (high) or `R_DrawColumnLow` (low). Temporarily redirected to `fuzzcolfunc` or `transcolfunc` for special effects. |
| `void (*basecolfunc)(void)` | `basecolfunc` | The base (non-special) column function, used to restore `colfunc` after drawing fuzz columns. |
| `void (*fuzzcolfunc)(void)` | `fuzzcolfunc` | Column renderer for the fuzz (Spectre) effect. Always `R_DrawFuzzColumn`. |
| `void (*spanfunc)(void)` | `spanfunc` | Floor/ceiling horizontal span renderer. `R_DrawSpan` (high) or `R_DrawSpanLow` (low). |

Note: `transcolfunc` (for color-translated sprites) is declared in `r_main.c` but not re-declared in this header.

---

## Functions

### Geometry Utilities

```c
int R_PointOnSide(fixed_t x, fixed_t y, node_t* node)
```
Returns `0` if `(x,y)` is on the front side of BSP `node`, `1` for back side. Used during BSP traversal.

---

```c
int R_PointOnSegSide(fixed_t x, fixed_t y, seg_t* line)
```
Returns `0` if `(x,y)` is on the front side of seg `line`, `1` for back side. Used for sprite ordering.

---

```c
angle_t R_PointToAngle(fixed_t x, fixed_t y)
```
Returns the binary view angle from the current viewpoint `(viewx, viewy)` to world point `(x, y)`. Uses octant decomposition and a precomputed `tantoangle[]` table.

---

```c
angle_t R_PointToAngle2(fixed_t x1, fixed_t y1, fixed_t x2, fixed_t y2)
```
Returns the binary angle from `(x1,y1)` to `(x2,y2)`. Side-effect: overwrites `viewx`/`viewy`.

---

```c
fixed_t R_PointToDist(fixed_t x, fixed_t y)
```
Returns the Euclidean distance from `(viewx, viewy)` to `(x, y)` in fixed-point map units.

---

```c
fixed_t R_ScaleFromGlobalAngle(angle_t visangle)
```
Returns the perspective scale factor for a wall column at `visangle`. Requires `rw_distance` and `rw_normalangle` to be set. Clamped to `[256, 64*FRACUNIT]`.

---

```c
subsector_t* R_PointInSubsector(fixed_t x, fixed_t y)
```
Walks the BSP tree to find the leaf subsector containing `(x, y)`.

---

```c
void R_AddPointToBox(int x, int y, fixed_t* box)
```
Expands the 4-element bounding box `box` to include point `(x, y)`.

---

### Refresh Control Functions

```c
void R_RenderPlayerView(player_t* player)
```
Top-level frame render. Called by the game loop once per tic. Runs setup, BSP traversal, plane drawing, and masked drawing in sequence. See `r_main.c` documentation for the full pipeline.

---

```c
void R_Init(void)
```
One-time renderer startup. Initializes all renderer subsystems (data, tables, light maps, sky, translation tables). Called from `D_DoomMain`.

---

```c
void R_SetViewSize(int blocks, int detail)
```
Deferred view-size/detail change. Sets `setsizeneeded = true` and stores the request. Applied at the start of the next frame via `R_ExecuteSetViewSize`. Called from the menu system (`M_Responder`) when the player changes screen size or detail.

**Parameters:**
- `blocks` - Screen size 1-11 (11 = full screen, others scale by 32 pixels per block).
- `detail` - 0 = high detail, 1 = low detail.

---

## Data Structures

No structs or typedefs are defined in this header. It references `fixed_t`, `angle_t`, `lighttable_t`, `node_t`, `seg_t`, `subsector_t`, and `player_t` from other headers.

---

## Dependencies

| Header | Provides |
|--------|---------|
| `d_player.h` | `player_t` definition, needed for `R_RenderPlayerView` and `R_SetupFrame`. |
| `r_data.h` | `lighttable_t` typedef and texture/colormap declarations. |
