# File Overview

`p_map.c` is the collision detection and spatial interaction core of the DOOM engine. It implements all movement validation, wall-sliding, hitscan shooting, radius attacks, line-use detection, and sector-crush processing. This file is one of the most algorithmically important in the entire codebase.

## Architectural Role

DOOM's world geometry is organized into a BSP tree for rendering, but collision detection does not traverse the BSP tree directly. Instead, it uses two precomputed spatial acceleration structures:

- **BLOCKMAP**: a coarse 128x128-unit grid covering the entire map. Each cell stores a list of all `line_t` segments that intersect it and is the head of a linked list of all `mobj_t` objects whose origin falls within it. Collision queries are performed by enumerating only the relevant cells.
- **Subsector lookup**: `R_PointInSubsector(x, y)` uses the BSP tree to find which convex subsector (and thus which sector) contains a point, which gives the floor and ceiling heights at any position.

The movement validation algorithm (`P_CheckPosition` / `P_TryMove`) works as follows:
1. Determine which subsector contains the destination point to get base floor/ceiling heights.
2. Iterate all blockmap cells overlapping the mobj's bounding box. For each cell, call `P_BlockThingsIterator` with `PIT_CheckThing` to detect mobj-mobj overlaps.
3. Iterate the same cells with `P_BlockLinesIterator` and `PIT_CheckLine` to find all line segments that the bounding box crosses. Each crossed two-sided line narrows the usable floor-ceiling window.
4. If the remaining window is tall enough and the step height is acceptable, the move is committed.

Hitscan shooting uses `P_PathTraverse`, which performs a 2D DDA (Digital Differential Analyzer) ray march through the blockmap, collecting all line and thing intercepts in order of distance, then processes them nearest-first.

---

## Global Variables

### Temporary Movement State (shared across a single movement check)

| Type | Name | Description |
|------|------|-------------|
| `fixed_t[4]` | `tmbbox` | Axis-aligned bounding box of the moving object at the test position |
| `mobj_t*` | `tmthing` | The mobj currently being tested for movement |
| `int` | `tmflags` | Cached copy of `tmthing->flags` |
| `fixed_t` | `tmx` | Target X position being tested |
| `fixed_t` | `tmy` | Target Y position being tested |
| `boolean` | `floatok` | Set to `true` if the move would fit vertically (ceiling - floor >= height), even if rejected for other reasons. Exported via `p_local.h`. |
| `fixed_t` | `tmfloorz` | Highest floor height encountered during the position check |
| `fixed_t` | `tmceilingz` | Lowest ceiling height encountered |
| `fixed_t` | `tmdropoffz` | Lowest floor height on the "other side" of any contacted line; used to prevent monsters walking off ledges |
| `line_t*` | `ceilingline` | The line responsible for lowering `tmceilingz`; used by the sky-hack in `P_XYMovement` to prevent missiles exploding on sky-textured ceilings |
| `line_t*[8]` | `spechit` | Array of special lines crossed during the position check |
| `int` | `numspechit` | Count of entries in `spechit` |

### Slide Move State

| Type | Name | Description |
|------|------|-------------|
| `fixed_t` | `bestslidefrac` | Fraction along the trace at which the closest blocking line was hit |
| `fixed_t` | `secondslidefrac` | Fraction for the second-closest blocking line |
| `line_t*` | `bestslideline` | The closest wall being slid against |
| `line_t*` | `secondslideline` | Second-closest wall |
| `mobj_t*` | `slidemo` | The mobj currently performing a slide move |
| `fixed_t` | `tmxmove` | Adjusted X movement component after wall projection |
| `fixed_t` | `tmymove` | Adjusted Y movement component after wall projection |

### Hitscan / Shooting State

| Type | Name | Description |
|------|------|-------------|
| `mobj_t*` | `linetarget` | The mobj that was successfully aimed at or shot; exported for weapon code |
| `mobj_t*` | `shootthing` | The mobj firing the shot (cannot hit itself) |
| `fixed_t` | `shootz` | Z height of the shot origin (mid-body + 8 units) |
| `int` | `la_damage` | Damage to deal on hit; `0` means this is a test trace only |
| `fixed_t` | `attackrange` | Maximum range of the current attack |
| `fixed_t` | `aimslope` | The vertical slope of the successful aim (rise/run in fixed-point) |

### Use-Line State

| Type | Name | Description |
|------|------|-------------|
| `mobj_t*` | `usething` | The player mobj attempting to use a line |

### Radius Attack State

| Type | Name | Description |
|------|------|-------------|
| `mobj_t*` | `bombsource` | The creature that caused the explosion |
| `mobj_t*` | `bombspot` | The explosion's center object |
| `int` | `bombdamage` | Maximum damage at ground zero |

### Sector Change State

| Type | Name | Description |
|------|------|-------------|
| `boolean` | `crushchange` | If `true`, things that do not fit in the modified sector take crushing damage |
| `boolean` | `nofit` | Set to `true` if any thing could not fit after a sector height change |

---

## Functions

### `PIT_StompThing`

```c
boolean PIT_StompThing(mobj_t* thing)
```

**Purpose:** Block-things iterator callback used by `P_TeleportMove`. Inflicts 10,000 damage (instant kill) on any shootable mobj occupying the teleport destination.

**Parameters:**
- `thing` - Candidate mobj from the current blockmap cell.

**Return value:** `true` to continue iterating; `false` only if a non-player mobj on a non-boss-level map is in the way (prevents teleportation).

**Key logic:** Computes Manhattan-style overlap using combined radii. Skips self. Non-player mobjs can block teleportation on maps other than map 30.

---

### `P_TeleportMove`

```c
boolean P_TeleportMove(mobj_t* thing, fixed_t x, fixed_t y)
```

**Purpose:** Moves a mobj directly to a new position, killing anything in the way. Used by teleporter specials.

**Parameters:**
- `thing` - The mobj being teleported.
- `x`, `y` - Destination coordinates.

**Return value:** `true` if the teleport succeeded.

**Key logic:**
1. Sets up `tmthing`, `tmflags`, `tmbbox` for the destination.
2. Uses `R_PointInSubsector` to get base floor/ceiling at the destination.
3. Iterates nearby blockmap cells with `PIT_StompThing` to kill any occupants.
4. Unlinks from old position, updates coordinates and floor/ceiling, re-links at new position.

---

### `PIT_CheckLine`

```c
boolean PIT_CheckLine(line_t* ld)
```

**Purpose:** Block-lines iterator callback for `P_CheckPosition`. Tests whether a line segment blocks or restricts the movement of `tmthing`.

**Parameters:**
- `ld` - Candidate line from the current blockmap cell.

**Return value:** `true` to continue checking (line does not block); `false` if the line is an impassable obstacle.

**Key logic:**
1. Quick AABB rejection: if `tmbbox` does not overlap `ld->bbox`, returns `true` immediately.
2. `P_BoxOnLineSide`: if the bounding box is entirely on one side, it is not crossing the line.
3. One-sided lines always block (returns `false`).
4. For non-missiles: `ML_BLOCKING` always blocks; `ML_BLOCKMONSTERS` blocks non-player mobjs.
5. Calls `P_LineOpening` to compute `opentop`, `openbottom`, `openrange`, `lowfloor`.
6. Updates `tmceilingz`, `tmfloorz`, `tmdropoffz`, and `ceilingline` based on the opening.
7. Records special lines in `spechit` for later processing.

---

### `PIT_CheckThing`

```c
boolean PIT_CheckThing(mobj_t* thing)
```

**Purpose:** Block-things iterator callback for `P_CheckPosition`. Tests whether `tmthing` overlaps another mobj.

**Return value:** `true` to continue (no blocking collision); `false` to stop (collision occurred).

**Key logic:**
- Skips anything without `MF_SOLID`, `MF_SPECIAL`, or `MF_SHOOTABLE`.
- Skips self and non-overlapping objects (combined-radius test).
- **Skull-fly**: deals `(1-8)*damage` and stops skull movement.
- **Missiles**: checks vertical overlap (can fly over/under). Missiles from the same species pass through each other (with player exception). Damage is `(1-8)*info->damage`. Returns `false` (stop traversal).
- **Special items**: if `tmthing` has `MF_PICKUP`, calls `P_TouchSpecialThing`. Non-solid items do not block.
- **Solid things**: returns `!(thing->flags & MF_SOLID)` - only solid things block.

---

### `P_CheckPosition`

```c
boolean P_CheckPosition(mobj_t* thing, fixed_t x, fixed_t y)
```

**Purpose:** Informational position check. Determines whether a mobj could occupy position `(x, y)` and updates the shared movement state variables (`tmfloorz`, `tmceilingz`, `tmdropoffz`, `spechit`). Does not actually move anything.

**Parameters:**
- `thing` - The mobj to test.
- `x`, `y` - The position to test.

**Return value:** `true` if the position is unobstructed; `false` if blocked by a line or solid mobj.

**Key logic:**
1. Sets up `tmthing`, `tmbbox` at `(x, y)`.
2. `R_PointInSubsector` gives the base floor/ceiling from the subsector at `(x, y)`.
3. Returns immediately `true` if `MF_NOCLIP` is set.
4. First pass: thing-vs-thing check. The bounding box is expanded by `MAXRADIUS` when computing block range because mobj origins can be in a different cell than their body extent.
5. Second pass: line check (no radius expansion needed since lines are stored in every cell they pass through).

---

### `P_TryMove`

```c
boolean P_TryMove(mobj_t* thing, fixed_t x, fixed_t y)
```

**Purpose:** Attempts to move `thing` to `(x, y)`. This is the primary movement function used by `P_XYMovement`. Unlike `P_CheckPosition`, it actually commits the position change if valid.

**Parameters:**
- `thing` - The mobj to move.
- `x`, `y` - Target position.

**Return value:** `true` if the move succeeded; `false` if blocked.

**Key logic:**
1. Calls `P_CheckPosition`; bails if it returns `false`.
2. Checks that `tmceilingz - tmfloorz >= thing->height` (gap must fit the mobj).
3. Sets `floatok = true` at this point (gap fits, even if step is too high).
4. Checks ceiling descent: `tmceilingz - thing->z < thing->height` (would need to lower to fit).
5. Checks step height: `tmfloorz - thing->z > 24*FRACUNIT` (step too high to walk up).
6. Checks drop-off: non-flying, non-drop-off things refuse to walk over a ledge higher than 24 units.
7. Unlinks from old position, updates position and floor/ceiling cache, re-links.
8. Processes special line crossings: for each line in `spechit`, checks if the mobj actually crossed it (old side != new side) and calls `P_CrossSpecialLine`.

---

### `P_ThingHeightClip`

```c
boolean P_ThingHeightClip(mobj_t* thing)
```

**Purpose:** Adjusts a mobj's Z position and floor/ceiling cache after sector heights have changed (e.g., a moving floor). Called for all mobjs in blocks adjacent to the moving sector.

**Return value:** `false` if the thing no longer fits (ceiling - floor < height).

**Key logic:** Re-runs `P_CheckPosition` at the current XY to refresh floor/ceiling. If the thing was on the floor, it follows the floor. If floating, it is pushed down from the ceiling only if forced.

---

### `P_HitSlideLine`

```c
void P_HitSlideLine(line_t* ld)
```

**Purpose:** Projects the movement vector `(tmxmove, tmymove)` onto the blocking wall `ld`, computing the component of velocity that runs parallel to the wall.

**Key logic:**
- Horizontal/vertical lines: simply zero the perpendicular component.
- Angled lines: computes `lineangle` from the line's direction, `moveangle` from the momentum, finds the angular difference `deltaangle`, and scales the movement length by `cos(deltaangle)` using the fine-angle lookup table to get the projected speed.

---

### `PTR_SlideTraverse`

```c
boolean PTR_SlideTraverse(intercept_t* in)
```

**Purpose:** Path traversal callback for `P_SlideMove`. Tests each intercepted line to see whether it blocks the slide. Tracks the nearest and second-nearest blocking lines.

**Key logic:** One-sided lines always block. Two-sided lines block if the opening is too small, too low, or the step is too high. Updates `bestslidefrac`/`bestslideline` and `secondslidefrac`/`secondslideline`.

---

### `P_SlideMove`

```c
void P_SlideMove(mobj_t* mo)
```

**Purpose:** When a player's movement is blocked, attempts to slide along the wall rather than stopping dead. This implements the smooth wall-hugging behavior.

**Parameters:**
- `mo` - The player mobj.

**Key logic:**
1. Sets `slidemo = mo` and `bestslidefrac = FRACUNIT+1`.
2. Traces three rays from the three "leading corners" of the bounding box (the corner ahead in X, ahead in Y, and the diagonal corner) using `P_PathTraverse` with `PTR_SlideTraverse`.
3. If no blocking line was found (`bestslidefrac == FRACUNIT+1`), falls through to a "stairstep" fallback: tries moving only in Y, then only in X.
4. Otherwise, moves flush to the wall (up to `bestslidefrac - 0x800`), then projects the remaining momentum along the wall using `P_HitSlideLine` and tries to continue.
5. Limited to 3 attempts (`hitcount`) to prevent infinite loops.

---

### `PTR_AimTraverse`

```c
boolean PTR_AimTraverse(intercept_t* in)
```

**Purpose:** Path traversal callback for `P_AimLineAttack`. Tracks the narrowing vertical window as the trace passes through two-sided lines, and records the first mobj within that window as `linetarget`.

**Key logic:**
- Lines: narrows `topslope` and `bottomslope` based on the opening geometry. Stops if the window closes to zero.
- Things: computes slopes to the top and bottom of the thing; if both are within `[bottomslope, topslope]`, sets `aimslope` to the midpoint and `linetarget` to the thing.

---

### `PTR_ShootTraverse`

```c
boolean PTR_ShootTraverse(intercept_t* in)
```

**Purpose:** Path traversal callback for `P_LineAttack`. Performs the actual damage application for hitscan weapons.

**Key logic:**
- Lines with specials: calls `P_ShootSpecialLine`.
- One-sided lines: jumps to `hitline` which spawns a puff. Sky ceiling hack: if the puff would be above the sky ceiling, returns without spawning anything.
- Two-sided lines: checks floor/ceiling slopes to see if the shot passes through the opening; if so, continues traversal.
- Things: vertical slope check against `aimslope`. On hit, spawns blood or puff depending on `MF_NOBLOOD`. Calls `P_DamageMobj` if `la_damage > 0`.

---

### `P_AimLineAttack`

```c
fixed_t P_AimLineAttack(mobj_t* t1, angle_t angle, fixed_t distance)
```

**Purpose:** Performs an autoaim trace in the given direction. Sets `linetarget` and returns the vertical slope needed to hit the target.

**Parameters:**
- `t1` - The firing mobj.
- `angle` - Horizontal firing angle.
- `distance` - Maximum range.

**Return value:** The aim slope to pass to `P_LineAttack`; `0` if no target was found.

**Key logic:** Sets up `topslope = 100/160` and `bottomslope = -100/160` (the view frustum slopes in the original 320x200 resolution) then calls `P_PathTraverse` with `PTR_AimTraverse`. The DOOM autoaim system works by progressively narrowing this vertical window as two-sided lines are traversed.

---

### `P_LineAttack`

```c
void P_LineAttack(mobj_t* t1, angle_t angle, fixed_t distance,
                  fixed_t slope, int damage)
```

**Purpose:** Fires a hitscan attack in the given direction and slope. The primary weapon fire function for pistol, chaingun, and shotgun.

**Parameters:**
- `t1` - The firing mobj.
- `angle` - Horizontal angle.
- `distance` - Maximum range.
- `slope` - Vertical slope (from `P_AimLineAttack` or fixed value for inaccuracy).
- `damage` - Damage to deal on hit; `0` for a test trace.

**Key logic:** Sets global shooting state, computes the endpoint `(x2, y2)` from `finecosine`/`finesine` tables, then calls `P_PathTraverse` with `PTR_ShootTraverse`.

---

### `PTR_UseTraverse`

```c
boolean PTR_UseTraverse(intercept_t* in)
```

**Purpose:** Path traversal callback for `P_UseLines`. Processes each intercepted line to see if it is a special the player can activate.

**Key logic:** Non-special, passable lines continue traversal. Non-special, impassable lines play `sfx_noway` and stop. Special lines call `P_UseSpecialLine` and stop (only one special per use action).

---

### `P_UseLines`

```c
void P_UseLines(player_t* player)
```

**Purpose:** Called each tic when the player presses the Use key. Traces a short ray (64 units = `USERANGE`) in front of the player to find and activate interactive linedefs.

**Key logic:** Computes endpoint from player angle with `finecosine`/`finesine`, then calls `P_PathTraverse(PT_ADDLINES, PTR_UseTraverse)`.

---

### `PIT_RadiusAttack`

```c
boolean PIT_RadiusAttack(mobj_t* thing)
```

**Purpose:** Block-things iterator callback for `P_RadiusAttack`. Applies splash damage to each thing within blast radius.

**Key logic:**
- Immune: `MT_CYBORG` and `MT_SPIDER` (boss monsters take no splash damage).
- Distance: uses Chebyshev distance (max of |dx|, |dy|) minus the thing's radius.
- Line of sight: only damages if `P_CheckSight` succeeds (explosions do not wrap around walls).
- Damage: `bombdamage - dist` (scales linearly with distance).

---

### `P_RadiusAttack`

```c
void P_RadiusAttack(mobj_t* spot, mobj_t* source, int damage)
```

**Purpose:** Initiates a radius/splash damage attack (rockets, barrel explosions).

**Parameters:**
- `spot` - The explosion center mobj.
- `source` - The creature responsible (for frag/kill credit).
- `damage` - Maximum damage at ground zero.

**Key logic:** Computes the blockmap range from `(damage + MAXRADIUS)` converted to blocks, sets global bomb state, then iterates all blockmap cells in the range with `PIT_RadiusAttack`.

---

### `PIT_ChangeSector`

```c
boolean PIT_ChangeSector(mobj_t* thing)
```

**Purpose:** Block-things iterator callback for `P_ChangeSector`. Adjusts each thing's height after a floor or ceiling change.

**Key logic:**
- If `P_ThingHeightClip` succeeds, the thing fits and iteration continues.
- Dead things with no space become gibs (`S_GIBS` state), lose solidity, and have radius/height zeroed.
- Dropped items are removed.
- Non-shootable things are assumed to be already-killed gibs.
- Living shootable things: `nofit = true`. If `crushchange` is active and every 4th tic (`leveltime&3`), deals 10 damage and spawns a blood spray.

---

### `P_ChangeSector`

```c
boolean P_ChangeSector(sector_t* sector, boolean crunch)
```

**Purpose:** Called by floor/ceiling movers after adjusting a sector's height. Iterates all things in nearby blockmap cells to update their positions.

**Parameters:**
- `sector` - The sector whose height changed.
- `crunch` - If `true`, things that do not fit take crushing damage.

**Return value:** `true` if any thing could not fit (caller should reverse height change if not crushing).

**Key logic:** Uses `sector->blockbox` (precomputed block range for this sector) to enumerate cells, then calls `P_BlockThingsIterator` with `PIT_ChangeSector` for each.

---

## Data Structures

No new struct types are defined in this file. It operates on `mobj_t`, `line_t`, `sector_t`, `intercept_t`, and `divline_t` (all defined in headers it includes).

---

## Dependencies

| File | Why needed |
|------|-----------|
| `stdlib.h` | `abs()` for distance calculations |
| `m_bbox.h` | Bounding box constants (`BOXTOP`, `BOXBOTTOM`, `BOXLEFT`, `BOXRIGHT`) |
| `m_random.h` | `P_Random()` for damage randomization and skull-fly checks |
| `i_system.h` | `I_Error()` for fatal assertion failures |
| `doomdef.h` | Core types, constants |
| `p_local.h` | All play subsystem types and function prototypes |
| `s_sound.h` | `S_StartSound()` for `sfx_noway` in `PTR_UseTraverse` |
| `doomstat.h` | `gamemap`, `leveltime`, global game state |
| `r_state.h` | `R_PointInSubsector`, `skyflatnum`, `lines[]`, `sectors[]` |
| `sounds.h` | Sound effect enum values |
