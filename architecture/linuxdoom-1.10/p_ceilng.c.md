# File Overview

`p_ceilng.c` (note the intentional misspelling in the original source) implements all ceiling animation logic for the DOOM engine. This file is part of the "P" (play) subsystem and handles the thinker-driven movement of ceilings: raising, lowering, and crush-damaging anything caught between a moving ceiling and the floor. It manages an active ceiling registry so that in-stasis ceilings can be paused and reactivated by special line triggers.

Ceiling animations are driven by the thinker system: each active ceiling has a `ceiling_t` thinker whose action function is `T_MoveCeiling`, called once per game tic. The ceiling types range from simple one-shot raises/lowers to perpetually bouncing crushing ceilings.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `ceiling_t*[MAXCEILINGS]` | `activeceilings` | Registry of all currently active (and in-stasis) ceiling thinkers. Indexed 0..MAXCEILINGS-1; NULL slots are unused. |

---

## Functions

### `void T_MoveCeiling(ceiling_t* ceiling)`

**Signature:** `void T_MoveCeiling(ceiling_t* ceiling)`

**Purpose:** Per-tic thinker function for a moving ceiling. Called each game tic by the thinker system to advance the ceiling's position toward its destination.

**Parameters:**
- `ceiling` - Pointer to the `ceiling_t` state structure for this ceiling.

**Return value:** None.

**Key logic:**
- Switches on `ceiling->direction`: `0` = in stasis (no-op), `1` = moving up, `-1` = moving down.
- Calls `T_MovePlane()` to actually move the ceiling plane in the sector.
- Every 8 tics (when `!(leveltime&7)`), plays the `sfx_stnmov` stone-moving sound, except for `silentCrushAndRaise` ceilings.
- When `T_MovePlane()` returns `pastdest`:
  - `raiseToHighest`: Calls `P_RemoveActiveCeiling()` to remove the thinker.
  - `silentCrushAndRaise`: Plays `sfx_pstop`, falls through to reverse direction.
  - `fastCrushAndRaise` / `crushAndRaise`: Reverses direction (sets `direction = -1`).
  - `lowerAndCrush` / `lowerToFloor`: Calls `P_RemoveActiveCeiling()`.
- When moving down and `T_MovePlane()` returns `crushed` (something blocking):
  - `silentCrushAndRaise` / `crushAndRaise` / `lowerAndCrush`: Slows speed to `CEILSPEED/8` to grind slowly.

---

### `int EV_DoCeiling(line_t* line, ceiling_e type)`

**Signature:** `int EV_DoCeiling(line_t* line, ceiling_e type)`

**Purpose:** Event handler to initiate ceiling movement on all sectors tagged to the triggering line. This is the entry point called from the special-line processing code when a player or monster activates a ceiling trigger.

**Parameters:**
- `line` - The triggering linedef; its `tag` field identifies which sectors to affect.
- `type` - The `ceiling_e` enumeration value specifying what kind of ceiling movement to perform.

**Return value:** `1` if at least one ceiling was started, `0` if no sectors were affected.

**Key logic:**
- For `fastCrushAndRaise`, `silentCrushAndRaise`, and `crushAndRaise` types, first calls `P_ActivateInStasisCeiling(line)` to reactivate any previously paused ceilings of the same tag before creating new ones.
- Iterates all sectors tagged to `line->tag` via `P_FindSectorFromLineTag()`. Skips sectors that already have `specialdata` set (already animated).
- For each eligible sector, allocates a `ceiling_t` with `Z_Malloc(PU_LEVSPEC)`, links it as a thinker with `P_AddThinker`, and sets `sec->specialdata = ceiling`.
- Configures `ceiling->topheight`, `bottomheight`, `speed`, `crush`, and `direction` according to `type`:
  - `fastCrushAndRaise`: moves down at `CEILSPEED*2`, crush enabled, 8 units above floor.
  - `silentCrushAndRaise` / `crushAndRaise`: crush enabled, bounce between original ceiling height and 8 units above floor.
  - `lowerAndCrush` / `lowerToFloor`: move down (lowerToFloor goes all the way to floor; lowerAndCrush stops 8 units above).
  - `raiseToHighest`: moves up to the highest surrounding ceiling.
- Registers the ceiling with `P_AddActiveCeiling()`.

---

### `void P_AddActiveCeiling(ceiling_t* c)`

**Signature:** `void P_AddActiveCeiling(ceiling_t* c)`

**Purpose:** Registers a newly created ceiling thinker in the `activeceilings[]` array so it can be found by tag for stasis and stop operations.

**Parameters:**
- `c` - The ceiling thinker to register.

**Return value:** None.

**Key logic:** Linearly scans `activeceilings[0..MAXCEILINGS-1]` for the first `NULL` slot and stores `c` there. Silently fails (no error) if the array is full, which would be a bug.

---

### `void P_RemoveActiveCeiling(ceiling_t* c)`

**Signature:** `void P_RemoveActiveCeiling(ceiling_t* c)`

**Purpose:** Deactivates a ceiling thinker: clears the sector's `specialdata` pointer, removes the thinker from the global thinker list, and frees the slot in `activeceilings[]`.

**Parameters:**
- `c` - The ceiling thinker to remove.

**Return value:** None.

**Key logic:** Scans `activeceilings[]` for the matching pointer. When found:
1. Sets `activeceilings[i]->sector->specialdata = NULL` to mark the sector as free for new animations.
2. Calls `P_RemoveThinker(&activeceilings[i]->thinker)` to unlink and schedule the memory for freeing.
3. Sets the slot to `NULL`.

---

### `void P_ActivateInStasisCeiling(line_t* line)`

**Signature:** `void P_ActivateInStasisCeiling(line_t* line)`

**Purpose:** Reactivates ceiling thinkers that were previously put in stasis (paused) by `EV_CeilingCrushStop`, matching by tag.

**Parameters:**
- `line` - The triggering line; its `tag` is used to find matching stasis ceilings.

**Return value:** None.

**Key logic:** Scans all `activeceilings[]` entries. For each that:
- Is non-NULL,
- Has `tag` matching `line->tag`,
- Has `direction == 0` (in stasis):

Restores `direction` to `olddirection` and reinstalls `T_MoveCeiling` as the thinker action function.

---

### `int EV_CeilingCrushStop(line_t* line)`

**Signature:** `int EV_CeilingCrushStop(line_t* line)`

**Purpose:** Puts all active (non-stasis) crushing ceilings matching the line's tag into stasis, effectively pausing them.

**Parameters:**
- `line` - The triggering line; its `tag` selects which ceilings to pause.

**Return value:** `1` if at least one ceiling was stopped, `0` if none matched.

**Key logic:** Scans `activeceilings[]`. For each non-NULL entry with matching tag and `direction != 0`:
1. Saves `direction` into `olddirection`.
2. Sets the thinker function to `NULL` (acv cast), disabling per-tic calls.
3. Sets `direction = 0` to mark it as in-stasis.

---

## Data Structures

This file uses the `ceiling_t` structure (defined in `p_spec.h`/`r_defs.h`):

```c
typedef struct
{
    thinker_t   thinker;        // linked into thinker list
    ceiling_e   type;           // ceiling behavior type
    sector_t*   sector;         // affected sector
    fixed_t     bottomheight;   // lower travel limit
    fixed_t     topheight;      // upper travel limit
    fixed_t     speed;          // units per tic
    boolean     crush;          // if true, damages things
    int         direction;      // 1=up, 0=stasis, -1=down
    int         tag;            // sector tag for matching
    int         olddirection;   // saved direction during stasis
} ceiling_t;
```

The `ceiling_e` enum (defined elsewhere) includes:
- `lowerToFloor`, `raiseToHighest`, `lowerAndCrush`
- `crushAndRaise`, `fastCrushAndRaise`, `silentCrushAndRaise`

The `result_e` enum (from `p_floor.c` / `p_spec.h`):
- `ok` - Move succeeded normally.
- `crushed` - Move was blocked by a thing being crushed.
- `pastdest` - Reached the destination height.

---

## Dependencies

| File | Purpose |
|------|---------|
| `z_zone.h` | `Z_Malloc()` for allocating ceiling thinker structures |
| `doomdef.h` | Core type definitions, `MAXCEILINGS`, `CEILSPEED` |
| `p_local.h` | `T_MovePlane()`, `P_AddThinker()`, `P_RemoveThinker()`, `P_FindSectorFromLineTag()`, `P_FindHighestCeilingSurrounding()`, sector/line types |
| `s_sound.h` | `S_StartSound()` for ceiling movement sounds |
| `doomstat.h` | `leveltime` global |
| `r_state.h` | `sectors[]` array |
| `sounds.h` | Sound effect enum constants (`sfx_stnmov`, `sfx_pstop`) |
