# File Overview

`am_map.c` implements the DOOM automap — the overhead map display that overlays the screen when the player presses TAB. It handles all rendering of the map view including walls, players, things (enemies/objects), grid lines, and mark points. The automap operates in its own coordinate space (map coordinates) separate from the screen's framebuffer coordinates, and provides smooth panning, zooming, and player-following behaviour. It also implements the `IDDT` cheat code that reveals all lines and things on the map.

The automap is one of the most self-contained modules in the engine, directly writing pixels into the screen framebuffer using a software Bresenham line-drawing algorithm.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `boolean` | `automapactive` | Exported flag; true when the automap is displayed instead of the 3D view. |

### Static (module-private) variables

| Type | Name | Description |
|------|------|-------------|
| `int` | `cheating` | Cheat level: 0 = normal, 1 = all lines visible, 2 = all lines + things visible. Cycles via `IDDT`. |
| `int` | `grid` | Non-zero when the background grid is displayed. |
| `int` | `leveljuststarted` | Kluge flag; prevents automap from running before `AM_LevelInit` has been called. |
| `int` | `finit_width` | Initial framebuffer window width (`SCREENWIDTH`). |
| `int` | `finit_height` | Initial framebuffer window height (`SCREENHEIGHT - 32`, leaving room for status bar). |
| `int` | `f_x`, `f_y` | Top-left corner of the automap window on screen. |
| `int` | `f_w`, `f_h` | Width and height of the automap window on screen. |
| `int` | `lightlev` | Current light level used for the strobing colour effect (currently unused). |
| `byte*` | `fb` | Pointer to the pseudo-framebuffer (`screens[0]`). |
| `int` | `amclock` | Automap tick counter; incremented each `AM_Ticker` call. |
| `mpoint_t` | `m_paninc` | Per-tic pan increment in map coordinates. |
| `fixed_t` | `mtof_zoommul` | Per-tic zoom multiplier for map-to-framebuffer scaling. |
| `fixed_t` | `ftom_zoommul` | Per-tic zoom multiplier for framebuffer-to-map scaling. |
| `fixed_t` | `m_x`, `m_y` | Lower-left (origin) of the visible map window in map coordinates. |
| `fixed_t` | `m_x2`, `m_y2` | Upper-right corner of the visible map window in map coordinates. |
| `fixed_t` | `m_w`, `m_h` | Width and height of the visible map window in map coordinates. |
| `fixed_t` | `min_x`, `min_y`, `max_x`, `max_y` | Bounding box of all map vertices. |
| `fixed_t` | `max_w`, `max_h` | Map dimensions (max - min). |
| `fixed_t` | `min_w`, `min_h` | Minimum window dimensions (based on `PLAYERRADIUS`). |
| `fixed_t` | `min_scale_mtof` | Scale at which the entire map fits in the window (maximum zoom-out). |
| `fixed_t` | `max_scale_mtof` | Scale at which the view is zoomed to player size (maximum zoom-in). |
| `fixed_t` | `old_m_w`, `old_m_h`, `old_m_x`, `old_m_y` | Saved scale/location for restoring after "big" zoom mode. |
| `mpoint_t` | `f_oldloc` | Previous player position used to detect movement in follow mode. |
| `fixed_t` | `scale_mtof` | Current map-to-framebuffer scale factor. |
| `fixed_t` | `scale_ftom` | Current framebuffer-to-map scale factor (reciprocal of `scale_mtof`). |
| `player_t*` | `plr` | Pointer to the player whose position the map centres on. |
| `patch_t*` | `marknums[10]` | WAD patches for the digit graphics used to label map mark points. |
| `mpoint_t` | `markpoints[AM_NUMMARKPOINTS]` | Array of up to 10 user-placed mark points; `x == -1` means empty. |
| `int` | `markpointnum` | Index of the next mark point slot to be written. |
| `int` | `followplayer` | 1 = automap follows the player; 0 = free-roaming pan mode. |
| `cheatseq_t` | `cheat_amap` | Cheat sequence descriptor for `IDDT`. |
| `boolean` | `stopped` | True when the automap is not running (fully shut down). |

### Vector graphics data

| Name | Description |
|------|-------------|
| `mline_t player_arrow[]` | 7 line segments forming an arrow shape to represent the player in normal mode. |
| `mline_t cheat_player_arrow[]` | 16 line segments forming "DDT" arrow visible when `IDDT` is active. |
| `mline_t triangle_guy[]` | 3-segment equilateral triangle used for enemies (unused in the final draw path). |
| `mline_t thintriangle_guy[]` | 3-segment thin triangle used to render thing positions in cheat mode 2. |

---

## Functions

### `AM_getIslope`
```c
void AM_getIslope(mline_t* ml, islope_t* is)
```
Calculates the slope and inverse slope of a map-coordinate line segment.
- **ml**: Input line segment in map coordinates.
- **is**: Output slope struct; `slp` = dy/dx, `islp` = dx/dy. Uses `MAXINT`/`-MAXINT` for vertical/horizontal lines.

---

### `AM_activateNewScale`
```c
void AM_activateNewScale(void)
```
Recomputes the map window's position and dimensions after a scale change, keeping the view centred on the current midpoint.

---

### `AM_saveScaleAndLoc`
```c
void AM_saveScaleAndLoc(void)
```
Saves the current map window position and dimensions into `old_m_*` variables for later restoration.

---

### `AM_restoreScaleAndLoc`
```c
void AM_restoreScaleAndLoc(void)
```
Restores the map window from `old_m_*` variables. If in follow-player mode, the position is recalculated around the player rather than restored literally. Recomputes `scale_mtof` and `scale_ftom`.

---

### `AM_addMark`
```c
void AM_addMark(void)
```
Places a mark point at the current centre of the automap window. Stores in `markpoints[markpointnum]` and advances `markpointnum` cyclically over `AM_NUMMARKPOINTS` slots.

---

### `AM_findMinMaxBoundaries`
```c
void AM_findMinMaxBoundaries(void)
```
Iterates all map vertices to find the level's bounding box (`min_x`, `min_y`, `max_x`, `max_y`). Computes `min_scale_mtof` (the scale that fits the whole level) and `max_scale_mtof` (the scale that fits a player-sized area).

---

### `AM_changeWindowLoc`
```c
void AM_changeWindowLoc(void)
```
Applies the per-tic pan increment (`m_paninc`) to `m_x`/`m_y`, clamping so the viewport centre stays within the level's bounding box. Switches to free-pan mode (sets `followplayer = 0`) when panning.

---

### `AM_initVariables`
```c
void AM_initVariables(void)
```
Initialises all runtime automap state when the automap is opened: sets `automapactive = true`, resets pan/zoom, finds a valid player to follow, centres the window on that player, saves the initial state, and notifies the status bar via `ST_Responder`.

---

### `AM_loadPics`
```c
void AM_loadPics(void)
```
Loads the ten mark-number patches (`AMMNUM0`–`AMMNUM9`) from the WAD into `marknums[]` as `PU_STATIC`.

---

### `AM_unloadPics`
```c
void AM_unloadPics(void)
```
Downgrades the ten mark-number patches from `PU_STATIC` to `PU_CACHE`, allowing them to be freed when memory is needed.

---

### `AM_clearMarks`
```c
void AM_clearMarks(void)
```
Clears all mark points by setting their `x` field to -1, and resets `markpointnum` to 0.

---

### `AM_LevelInit`
```c
void AM_LevelInit(void)
```
Called once per level when the automap is first opened. Resets `leveljuststarted`, sets the framebuffer window to cover the full screen minus the status bar, clears marks, finds the level boundaries, and calculates an initial scale that shows approximately 70% of the level.

---

### `AM_Stop`
```c
void AM_Stop(void)
```
Shuts down the automap: unloads mark patches, sets `automapactive = false`, notifies the status bar with `AM_MSGEXITED`, and sets `stopped = true`.

---

### `AM_Start`
```c
void AM_Start(void)
```
Opens the automap. Calls `AM_Stop` first if it was already running. Re-runs `AM_LevelInit` whenever the map or episode has changed since the last start. Calls `AM_initVariables` and `AM_loadPics`.

---

### `AM_minOutWindowScale`
```c
void AM_minOutWindowScale(void)
```
Zooms out to the minimum scale (showing the whole level) and applies the new scale.

---

### `AM_maxOutWindowScale`
```c
void AM_maxOutWindowScale(void)
```
Zooms in to the maximum scale (smallest possible window) and applies the new scale.

---

### `AM_Responder`
```c
boolean AM_Responder(event_t* ev)
```
Handles all keyboard events while the automap is open (and TAB to open it when closed). Returns `true` if the event was consumed.

Key bindings handled:
- Arrow keys: pan (only in free-pan mode).
- `=` / `-`: zoom in / zoom out.
- `TAB`: close automap.
- `0`: toggle "big" (full-level) zoom, saving/restoring position.
- `f`: toggle follow-player mode.
- `g`: toggle grid.
- `m`: place a mark point.
- `c`: clear all mark points.
- `IDDT` sequence: cycle `cheating` through 0→1→2→0.

---

### `AM_changeWindowScale`
```c
void AM_changeWindowScale(void)
```
Applies the current zoom multipliers (`mtof_zoommul`, `ftom_zoommul`) to `scale_mtof`/`scale_ftom` each tic, clamping to the min/max zoom limits.

---

### `AM_doFollowPlayer`
```c
void AM_doFollowPlayer(void)
```
If the player has moved since the last frame, re-centres the map window on the player's position. The position is snapped through MTOF/FTOM to avoid half-pixel drift.

---

### `AM_updateLightLev`
```c
void AM_updateLightLev(void)
```
Steps through a pre-defined table of light levels on a 6-tic cycle, updating `lightlev` for the strobing colour effect. Currently commented out in `AM_Ticker`.

---

### `AM_Ticker`
```c
void AM_Ticker(void)
```
Called once per game tic while the automap is active. Increments `amclock`, calls `AM_doFollowPlayer` if following, applies zoom changes, and applies pan changes.

---

### `AM_clearFB`
```c
void AM_clearFB(int color)
```
Fills the entire automap framebuffer region with a solid colour using `memset`. Used to paint the black background each frame.
- **color**: Palette index to fill with (normally `BLACK = 0`).

---

### `AM_clipMline`
```c
boolean AM_clipMline(mline_t* ml, fline_t* fl)
```
Clips a map-coordinate line segment to the visible automap window using a Cohen-Sutherland variant, then converts surviving endpoints to framebuffer coordinates via `CXMTOF`/`CYMTOF`. Writes the clipped framebuffer line to `*fl`.
- **ml**: Input line in map coordinates.
- **fl**: Output line in framebuffer coordinates.
- **Returns**: `true` if any portion of the line is visible; `false` if trivially rejected.

---

### `AM_drawFline`
```c
void AM_drawFline(fline_t* fl, int color)
```
Draws a single framebuffer-coordinate line segment using an integer Bresenham algorithm, writing directly to `fb`. Has a bounds-check debug path that prints to stderr (but does not crash) if coordinates are out of range.
- **fl**: Line endpoints in framebuffer pixel coordinates.
- **color**: Palette index.

---

### `AM_drawMline`
```c
void AM_drawMline(mline_t* ml, int color)
```
Clips a map-coordinate line and, if visible, draws it to the framebuffer. Combines `AM_clipMline` and `AM_drawFline`.

---

### `AM_drawGrid`
```c
void AM_drawGrid(int color)
```
Draws vertical and horizontal grid lines aligned to the blockmap grid (`MAPBLOCKUNITS`). Lines start at the first grid boundary inside the current view and step by one block unit.

---

### `AM_drawWalls`
```c
void AM_drawWalls(void)
```
Iterates all linedefs and draws those that are visible according to the following rules:
- If `cheating` or the line has `ML_MAPPED`: draw with colour depending on type (solid wall, floor-height change, ceiling-height change, two-sided transparent, teleporter, secret).
- If the player has the Computer Area Map power-up: draw unmapped lines in gray.
- Linedefs with `ML_DONTDRAW` (= `LINE_NEVERSEE`) are skipped unless cheating.

---

### `AM_rotate`
```c
void AM_rotate(fixed_t* x, fixed_t* y, angle_t a)
```
Rotates a 2D point `(x, y)` by angle `a` using the fine-angle sine/cosine tables. Used to orient the player arrow.

---

### `AM_drawLineCharacter`
```c
void AM_drawLineCharacter(mline_t* lineguy, int lineguylines, fixed_t scale, angle_t angle, int color, fixed_t x, fixed_t y)
```
Draws a vector-graphic "character" (player arrow, triangle, etc.) defined as an array of map-space line segments. Each segment is optionally scaled, optionally rotated, and then translated to `(x, y)` before being drawn via `AM_drawMline`.
- **lineguy**: Array of line segments defining the shape.
- **lineguylines**: Number of segments.
- **scale**: Fixed-point scale factor (0 = no scaling).
- **angle**: Rotation angle (0 = no rotation).
- **color**: Palette index.
- **x, y**: World-space translation.

---

### `AM_drawPlayers`
```c
void AM_drawPlayers(void)
```
Draws player arrows. In single-player: draws the normal or cheat arrow for the console player in white. In netgame: draws all active players with their team colours (green, gray, brown, red), drawing invisible players in near-black.

---

### `AM_drawThings`
```c
void AM_drawThings(int colors, int colorrange)
```
Iterates all sectors and draws a thin-triangle icon for every `mobj_t` on each sector's thing list. Only called when `cheating == 2`.

---

### `AM_drawMarks`
```c
void AM_drawMarks(void)
```
Draws the numeric mark-point patches at each active mark location, provided the mark is within the visible framebuffer area.

---

### `AM_drawCrosshair`
```c
void AM_drawCrosshair(int color)
```
Draws a single-pixel crosshair at the centre of the automap framebuffer.

---

### `AM_Drawer`
```c
void AM_Drawer(void)
```
Main automap rendering function, called once per display frame. Clears the framebuffer, optionally draws the grid, draws all walls, draws all players, optionally draws things (cheat level 2), draws the crosshair, draws mark-point numbers, and calls `V_MarkRect` to mark the region as dirty for blitting.

---

## Data Structures

### `fpoint_t`
```c
typedef struct { int x, y; } fpoint_t;
```
A 2D point in framebuffer (pixel) coordinates.

### `fline_t`
```c
typedef struct { fpoint_t a, b; } fline_t;
```
A line segment in framebuffer coordinates.

### `mpoint_t`
```c
typedef struct { fixed_t x, y; } mpoint_t;
```
A 2D point in map (fixed-point world) coordinates.

### `mline_t`
```c
typedef struct { mpoint_t a, b; } mline_t;
```
A line segment in map coordinates.

### `islope_t`
```c
typedef struct { fixed_t slp, islp; } islope_t;
```
Slope (`dy/dx`) and inverse slope (`dx/dy`) of a line, used by the clipping code.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `z_zone.h` | Memory zone tags (`PU_STATIC`, `PU_CACHE`) for mark number patches. |
| `doomdef.h` | Game constants (`SCREENWIDTH`, `SCREENHEIGHT`, `FRACUNIT`, `MAXINT`, etc.). |
| `st_stuff.h` | `ST_Responder` for status bar notifications on map open/close. |
| `p_local.h` | `PLAYERRADIUS`, `MAPBLOCKUNITS`, `bmaporgx`/`bmaporgy`, blockmap globals. |
| `w_wad.h` | `W_CacheLumpName` for loading `AMMNUM*` patches. |
| `m_cheat.h` | `cht_CheckCheat`, `cheatseq_t` for the `IDDT` cheat. |
| `i_system.h` | `I_Error` (indirectly via other headers). |
| `v_video.h` | `screens[]` framebuffer array, `V_MarkRect`, `V_DrawPatch`. |
| `doomstat.h` | `players[]`, `playeringame[]`, `consoleplayer`, `netgame`, `deathmatch`, etc. |
| `r_state.h` | `vertexes`, `numvertexes`, `lines`, `numlines`, `sectors`, `numsectors`. |
| `dstrings.h` | Automap UI strings (`AMSTR_FOLLOWON`, `AMSTR_GRIDON`, etc.). |
| `am_map.h` | Own header (message codes `AM_MSGENTERED`/`AM_MSGEXITED`, public function declarations). |
