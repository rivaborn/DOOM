# File Overview

`p_doors.c` implements all vertical door animation logic for the DOOM engine. Doors in DOOM are sector-based: the ceiling of a sector is raised or lowered to simulate a door opening or closing. This file handles the full lifecycle of vertical door thinkers, including normal doors, fast "blaze" doors, locked doors requiring key cards or skull keys, and timed doors (close-then-open, raise-after-delay). A large block of sliding door code is included but compiled out (`#if 0`), representing an abandoned feature.

---

## Global Variables

None declared at file scope in the active code. (The `slideFrameNames` and `slideFrames` arrays exist inside `#if 0` blocks and are not compiled.)

---

## Functions

### `void T_VerticalDoor(vldoor_t* door)`

**Signature:** `void T_VerticalDoor(vldoor_t* door)`

**Purpose:** Per-tic thinker function for a vertical door. Called each game tic to advance the door's ceiling toward its destination and handle state transitions (waiting at top, reversing, completing).

**Parameters:**
- `door` - Pointer to the `vldoor_t` state describing this door's current state.

**Return value:** None.

**Key logic:** Switches on `door->direction`:

- **`0` (WAITING):** Decrements `door->topcountdown`. When it reaches 0:
  - `blazeRaise` / `normal`: Start closing (direction = -1), play close sound.
  - `close30ThenOpen`: Start opening (direction = 1), play open sound.

- **`2` (INITIAL WAIT):** Decrements `door->topcountdown`. When 0 and type is `raiseIn5Mins`: switches to `direction = 1`, changes type to `normal`, plays open sound.

- **`-1` (DOWN / CLOSING):** Calls `T_MovePlane()` to lower the ceiling to `sector->floorheight`. On `pastdest`:
  - `blazeRaise` / `blazeClose`: Clears `specialdata`, removes thinker, plays blaze-close sound.
  - `normal` / `close`: Clears `specialdata`, removes thinker.
  - `close30ThenOpen`: Sets direction to 0 (wait), sets countdown to 35*30 tics (~30 seconds).
  On `crushed` (something blocking the door from closing):
  - `blazeClose` / `close`: Does nothing (the door is forced closed regardless).
  - All others: Reverses direction to 1 (open), plays open sound.

- **`1` (UP / OPENING):** Calls `T_MovePlane()` to raise the ceiling to `door->topheight`. On `pastdest`:
  - `blazeRaise` / `normal`: Sets `direction = 0`, starts `topcountdown` from `topwait`.
  - `close30ThenOpen` / `blazeOpen` / `open`: Clears `specialdata`, removes thinker (stays open permanently).

---

### `int EV_DoLockedDoor(line_t* line, vldoor_e type, mobj_t* thing)`

**Signature:** `int EV_DoLockedDoor(line_t* line, vldoor_e type, mobj_t* thing)`

**Purpose:** Attempts to open a tagged door that requires a key card or skull key. Displays a message and plays a sound if the activating player lacks the required key.

**Parameters:**
- `line` - The triggering linedef; its `special` number identifies the required key color.
- `type` - The door behavior type to use if access is granted.
- `thing` - The entity attempting to use the door (must be a player).

**Return value:** `0` if the player lacks the key or is not a player; otherwise returns the result of `EV_DoDoor()`.

**Key logic:**
- Returns 0 immediately if `thing->player` is NULL (non-player cannot use locked doors).
- Checks `line->special` to determine required key:
  - `99`, `133`: Requires blue card OR blue skull (`it_bluecard` or `it_blueskull`). Failure sets `player->message = PD_BLUEO` and plays `sfx_oof`.
  - `134`, `135`: Requires red card OR red skull. Failure: `PD_REDO`.
  - `136`, `137`: Requires yellow card OR yellow skull. Failure: `PD_YELLOWO`.
- On success, delegates to `EV_DoDoor(line, type)`.

---

### `int EV_DoDoor(line_t* line, vldoor_e type)`

**Signature:** `int EV_DoDoor(line_t* line, vldoor_e type)`

**Purpose:** Initiates door movement on all sectors tagged to the triggering line. Creates a `vldoor_t` thinker for each eligible sector.

**Parameters:**
- `line` - The triggering linedef.
- `type` - The door behavior type.

**Return value:** `1` if at least one door was started, `0` otherwise.

**Key logic:**
- Iterates all sectors matching `line->tag` via `P_FindSectorFromLineTag()`. Skips any sector already having `specialdata`.
- For each eligible sector: allocates `vldoor_t` with `Z_Malloc(PU_LEVSPEC)`, links it as a thinker, assigns `T_VerticalDoor` as the action function.
- Sets defaults: `topwait = VDOORWAIT`, `speed = VDOORSPEED`.
- Configures per type:
  - `blazeClose`: `topheight = P_FindLowestCeilingSurrounding - 4`, direction -1, speed x4.
  - `close`: Same as blazeClose but normal speed; plays `sfx_dorcls`.
  - `close30ThenOpen`: `topheight = current ceiling`, direction -1.
  - `blazeRaise` / `blazeOpen`: direction 1, speed x4, topheight = lowest surrounding ceiling - 4.
  - `normal` / `open`: direction 1, normal speed, topheight = lowest surrounding ceiling - 4.

---

### `void EV_VerticalDoor(line_t* line, mobj_t* thing)`

**Signature:** `void EV_VerticalDoor(line_t* line, mobj_t* thing)`

**Purpose:** Opens a door that is activated by direct player use (pressing the use key on the linedef), with no tag/sector scan needed. This handles the "manual" door case where the back sector of the used line is the door sector.

**Parameters:**
- `line` - The linedef that was used.
- `thing` - The entity that used the line.

**Return value:** None.

**Key logic:**
- Performs key checks based on `line->special` for lines 26/32 (blue), 27/34 (yellow), 28/33 (red). If the player lacks the key, sets a message and plays `sfx_oof`, then returns.
- Determines door sector: `sec = sides[line->sidenum[1]].sector` (back side).
- If `sec->specialdata` is already set (door already has a thinker), handles toggling for certain line specials (1, 26, 27, 28, 117): if closing, reverses to opening; if opening, reverses to closing (but only if the activator is a player - monsters never close doors this way).
- Otherwise creates a new `vldoor_t` thinker, setting direction to 1 (up) and configuring type based on `line->special`:
  - 1, 26, 27, 28: `normal` (raises then lowers after wait).
  - 31, 32, 33, 34: `open` (raises and stays open, clears `line->special` so it triggers only once).
  - 117: `blazeRaise` (fast raise/lower).
  - 118: `blazeOpen` (fast raise, stays open).
- Sets `topheight = P_FindLowestCeilingSurrounding(sec) - 4*FRACUNIT`.

---

### `void P_SpawnDoorCloseIn30(sector_t* sec)`

**Signature:** `void P_SpawnDoorCloseIn30(sector_t* sec)`

**Purpose:** Spawns a door thinker that will close after 30 seconds. Used for level-start door specials (sector type handling).

**Parameters:**
- `sec` - The sector whose ceiling is the door.

**Return value:** None.

**Key logic:** Creates a `vldoor_t` with direction 0 (waiting), type `normal`, and `topcountdown = 30 * 35` (30 seconds at 35 tics/sec). When the countdown expires, `T_VerticalDoor` will begin closing.

---

### `void P_SpawnDoorRaiseIn5Mins(sector_t* sec, int secnum)`

**Signature:** `void P_SpawnDoorRaiseIn5Mins(sector_t* sec, int secnum)`

**Purpose:** Spawns a door thinker that remains closed for 5 minutes and then opens. Used for certain map secrets or timed events.

**Parameters:**
- `sec` - The sector whose ceiling is the door.
- `secnum` - The sector number (unused in the current implementation).

**Return value:** None.

**Key logic:** Creates a `vldoor_t` with `direction = 2` (initial wait state), type `raiseIn5Mins`, `topcountdown = 5 * 60 * 35` (10,500 tics). Sets `topheight = P_FindLowestCeilingSurrounding(sec) - 4*FRACUNIT`.

---

## Data Structures

This file uses `vldoor_t` (defined in `p_spec.h`):

```c
typedef struct
{
    thinker_t   thinker;        // linked into thinker list
    vldoor_e    type;           // door behavior type
    sector_t*   sector;         // the door sector
    fixed_t     topheight;      // fully open position
    fixed_t     speed;          // movement speed (units/tic)
    int         direction;      // 1=up, 0=wait, -1=down, 2=initial wait
    int         topwait;        // tics to wait at top (total)
    int         topcountdown;   // countdown timer at top
} vldoor_t;
```

The `vldoor_e` enum includes:
- `normal` - Opens, waits, closes.
- `close` - Closes and stays closed.
- `open` - Opens and stays open.
- `raiseIn5Mins` - Opens after 5-minute delay.
- `blazeRaise` - Fast open, wait, close.
- `blazeOpen` - Fast open, stay open.
- `blazeClose` - Fast close.
- `close30ThenOpen` - Closes, waits 30 seconds, then opens.

Constants used:
- `VDOORSPEED` - Standard door speed.
- `VDOORWAIT` - Standard wait time at top.

---

## Dependencies

| File | Purpose |
|------|---------|
| `z_zone.h` | `Z_Malloc()` for allocating door thinker structures |
| `doomdef.h` | Core types, `VDOORSPEED`, `VDOORWAIT` constants |
| `p_local.h` | `T_MovePlane()`, `P_AddThinker()`, `P_RemoveThinker()`, `P_FindSectorFromLineTag()`, `P_FindLowestCeilingSurrounding()` |
| `s_sound.h` | `S_StartSound()` for door sounds |
| `doomstat.h` | Game state (game mode, net game flags) |
| `r_state.h` | `sectors[]`, `sides[]` arrays |
| `dstrings.h` | Locked-door player message strings (`PD_BLUEO`, etc.) |
| `sounds.h` | Sound effect constants (`sfx_doropn`, `sfx_dorcls`, `sfx_bdopn`, `sfx_bdcls`, `sfx_oof`) |
