# File Overview

`p_floor.c` implements floor and staircase animation for the DOOM engine. It provides:
1. `T_MovePlane` - The fundamental plane-movement primitive used by all moving sectors (floors, ceilings, doors, platforms).
2. `T_MoveFloor` - The per-tic thinker for a moving floor.
3. `EV_DoFloor` - The event handler that creates floor-movement thinkers in response to line triggers.
4. `EV_BuildStairs` - The staircase-building algorithm that cascades floor rises through connected sectors sharing the same floor texture.

---

## Global Variables

None at file scope. All movement state is carried in `floormove_t` structures allocated per-sector.

---

## Functions

### `result_e T_MovePlane(sector_t* sector, fixed_t speed, fixed_t dest, boolean crush, int floorOrCeiling, int direction)`

**Signature:** `result_e T_MovePlane(sector_t* sector, fixed_t speed, fixed_t dest, boolean crush, int floorOrCeiling, int direction)`

**Purpose:** The fundamental plane-movement primitive. Moves either the floor or ceiling of a sector by `speed` units per call toward `dest`, respecting crush damage logic. Called by floor thinkers, ceiling thinkers, door thinkers, and platform thinkers.

**Parameters:**
- `sector` - The sector whose floor or ceiling is being moved.
- `speed` - Distance (in fixed-point map units) to move this tic.
- `dest` - Target height to reach.
- `crush` - If true, damages things caught between the moving plane and the opposite plane.
- `floorOrCeiling` - `0` = move the floor, `1` = move the ceiling.
- `direction` - `-1` = downward, `1` = upward.

**Return value:** `result_e` enum:
- `ok` - Moved without reaching destination and nothing was crushed.
- `crushed` - A thing was crushed (could not fit after the move).
- `pastdest` - Reached or passed the destination height this tic.

**Key logic:**

For **floor movement (floorOrCeiling = 0)**:
- **Down (direction -1):** If moving down would overshoot `dest`, snaps to `dest` exactly, calls `P_ChangeSector()`. If `P_ChangeSector()` reports a crush, restores the previous height and re-calls `P_ChangeSector`. Returns `pastdest`.
  If no overshoot: subtracts `speed`, calls `P_ChangeSector()`. If crushed, restores height and returns `crushed`.
- **Up (direction 1):** Mirror logic. If would overshoot, snaps to dest. Otherwise adds speed; returns `crushed` if `crush` is enabled and something doesn't fit.

For **ceiling movement (floorOrCeiling = 1)**: Same structure as floor but operates on `sector->ceilingheight`. Note: ceiling upward movement has a commented-out crush-return block (`#if 0`) suggesting the behavior was intentionally not implemented for rising ceilings.

Returns `ok` if the tic completes without reaching the destination or causing a crush.

---

### `void T_MoveFloor(floormove_t* floor)`

**Signature:** `void T_MoveFloor(floormove_t* floor)`

**Purpose:** Per-tic thinker action for a moving floor. Advances the floor toward `floordestheight`, plays movement sounds, and handles completion.

**Parameters:**
- `floor` - The `floormove_t` thinker state.

**Return value:** None.

**Key logic:**
- Calls `T_MovePlane(floor->sector, floor->speed, floor->floordestheight, floor->crush, 0, floor->direction)`.
- Every 8 tics, plays `sfx_stnmov`.
- When `pastdest`:
  - Clears `sector->specialdata = NULL`.
  - If moving up and type is `donutRaise`: applies `floor->newspecial` and `floor->texture` to the sector.
  - If moving down and type is `lowerAndChange`: applies texture/special changes (floor adopts the texture of the lowest adjacent floor).
  - Calls `P_RemoveThinker()` to free the thinker.
  - Plays `sfx_pstop`.

---

### `int EV_DoFloor(line_t* line, floor_e floortype)`

**Signature:** `int EV_DoFloor(line_t* line, floor_e floortype)`

**Purpose:** Event handler that creates floor-movement thinkers for all sectors tagged to the triggering line. Supports a wide variety of floor behavior types.

**Parameters:**
- `line` - The triggering linedef.
- `floortype` - The `floor_e` enumeration specifying the desired floor behavior.

**Return value:** `1` if any floors were started, `0` if no tagged sectors were eligible.

**Key logic:**
Iterates all sectors matching `line->tag`. Skips sectors with existing `specialdata`. For each eligible sector, allocates a `floormove_t` thinker and configures it based on `floortype`:

| Floor Type | Direction | Speed | Destination |
|-----------|-----------|-------|-------------|
| `lowerFloor` | Down | FLOORSPEED | Highest surrounding floor |
| `lowerFloorToLowest` | Down | FLOORSPEED | Lowest surrounding floor |
| `turboLower` | Down | FLOORSPEED*4 | Highest surrounding floor (+8 if not at current) |
| `raiseFloor` | Up | FLOORSPEED | Lowest surrounding ceiling |
| `raiseFloorCrush` | Up | FLOORSPEED | Lowest surrounding ceiling - 8 units (with crush=true) |
| `raiseFloorTurbo` | Up | FLOORSPEED*4 | Next highest floor above current |
| `raiseFloorToNearest` | Up | FLOORSPEED | Next highest floor above current |
| `raiseFloor24` | Up | FLOORSPEED | Current floor + 24 units |
| `raiseFloor512` | Up | FLOORSPEED | Current floor + 512 units |
| `raiseFloor24AndChange` | Up | FLOORSPEED | Current + 24, copies trigger sector's floor texture/special |
| `raiseToTexture` | Up | FLOORSPEED | Current floor + minimum bottom texture height among adjacent two-sided lines |
| `lowerAndChange` | Down | FLOORSPEED | Lowest surrounding floor; copies texture/special from that neighbor |

---

### `int EV_BuildStairs(line_t* line, stair_e type)`

**Signature:** `int EV_BuildStairs(line_t* line, stair_e type)`

**Purpose:** Implements the "build stairs" special: raises a series of connected sectors into a staircase, each step being `stairsize` units higher than the previous.

**Parameters:**
- `line` - The triggering linedef.
- `type` - `build8` (slow, 8-unit steps) or `turbo16` (fast, 16-unit steps).

**Return value:** `1` if at least one sector was raised, `0` otherwise.

**Key logic (cascade algorithm):**
1. Finds all sectors tagged to `line->tag` as stair-start sectors.
2. For each start sector: creates a `floormove_t` thinker raising to `current floor + stairsize`.
3. Records the start sector's `floorpic` as the "stair texture" to propagate.
4. Inner loop: searches the current sector's two-sided lines for a connected back sector that:
   - Shares the same floor texture as the original start sector.
   - Does not already have `specialdata` (not already moving).
5. For each such neighbor: creates another `floormove_t` with `floordestheight` increased by another `stairsize`.
6. Advances to the neighbor sector and continues until no more matching neighbors are found.

Speed and step sizes:
- `build8`: `FLOORSPEED/4`, 8-unit steps.
- `turbo16`: `FLOORSPEED*4`, 16-unit steps.

---

## Data Structures

`floormove_t` (defined in `p_spec.h`):

```c
typedef struct
{
    thinker_t   thinker;
    floor_e     type;           // floor behavior type
    boolean     crush;          // damage things when they don't fit
    sector_t*   sector;         // affected sector
    int         direction;      // 1=up, -1=down
    int         newspecial;     // sector special to apply when complete
    short       texture;        // floor texture to apply when complete
    fixed_t     floordestheight;// destination floor height
    fixed_t     speed;          // movement speed (units/tic)
} floormove_t;
```

`floor_e` enum values include: `lowerFloor`, `lowerFloorToLowest`, `turboLower`, `raiseFloor`, `raiseFloorCrush`, `raiseFloor24`, `raiseFloor24AndChange`, `raiseToTexture`, `lowerAndChange`, `raiseFloor512`, `raiseFloorTurbo`, `raiseFloorToNearest`, `donutRaise`.

`stair_e` enum values: `build8`, `turbo16`.

`result_e` enum: `ok`, `crushed`, `pastdest`.

---

## Dependencies

| File | Purpose |
|------|---------|
| `z_zone.h` | `Z_Malloc()` for allocating floor thinker structures |
| `doomdef.h` | Core types, `FLOORSPEED`, `FRACUNIT`, `MAXINT` |
| `p_local.h` | `P_AddThinker()`, `P_RemoveThinker()`, `P_FindSectorFromLineTag()`, `P_FindLowestFloorSurrounding()`, `P_FindHighestFloorSurrounding()`, `P_FindLowestCeilingSurrounding()`, `P_FindNextHighestFloor()`, `P_ChangeSector()`, `twoSided()`, `getSide()`, `getSector()`, `getNextSector()` |
| `s_sound.h` | `S_StartSound()` for movement sounds |
| `doomstat.h` | `leveltime` |
| `r_state.h` | `sectors[]`, `sides[]`, `textureheight[]` |
| `sounds.h` | Sound constants (`sfx_stnmov`, `sfx_pstop`) |
