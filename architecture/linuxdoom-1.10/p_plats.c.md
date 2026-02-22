# File Overview

**Source file:** `linuxdoom-1.10/p_plats.c`

`p_plats.c` implements **elevator platforms** — the moving floor sectors that raise and lower in response to linedef triggers. Platforms are one of the most common interactive sector specials in DOOM maps: the classic "lower platform, player steps on, platform rises back up" mechanic is handled here.

A platform is a sector whose floor moves between a low point and a high point on a timed schedule. Each active platform is represented by a `plat_t` thinker that is updated every game tic. The file manages:

- The per-tic movement callback (`T_PlatRaise`) that drives each platform's state machine.
- The linedef-triggered activation function (`EV_DoPlat`) that creates new platform thinkers.
- A global registry of all active platforms (`activeplats[]`) that allows platforms to be stopped and restarted by tag.
- Utility functions for adding, removing, stopping (stasis), and reactivating platforms.

---

## Global Variables

| Type | Name | Purpose |
|------|------|---------|
| `plat_t*[MAXPLATS]` | `activeplats` | Registry of all currently active platform thinkers. Slots are NULL when unused. `MAXPLATS` is defined in `p_local.h` (30 in the original source). This array is searched linearly by tag when stopping or reactivating platforms. |

---

## Functions

### `T_PlatRaise`

```c
void T_PlatRaise(plat_t *plat)
```

**Purpose:** Per-tic thinker callback for a single platform. Implements the platform's three-state FSM (`up`, `down`, `waiting`) by delegating movement to `T_MovePlane` and handling state transitions and sound cues.

**Parameters:**
- `plat` - the platform thinker being updated.

**Key logic — state machine:**

**`up` state:**
- Calls `T_MovePlane(plat->sector, plat->speed, plat->high, plat->crush, 0, 1)` to move the floor upward toward `plat->high`.
- For `raiseAndChange` and `raiseToNearestAndChange` types, plays `sfx_stnmov` (stone move) every 8 tics.
- If `T_MovePlane` returns `crushed` and the platform is not a crushing type, reverses to `down` and plays `sfx_pstart`.
- If `T_MovePlane` returns `pastdest` (reached the target height):
  - Sets `status = waiting`, resets `count = plat->wait`, plays `sfx_pstop`.
  - For `blazeDWUS`, `downWaitUpStay`, `raiseAndChange`, `raiseToNearestAndChange`: calls `P_RemoveActivePlat` because these types only run once.

**`down` state:**
- Calls `T_MovePlane` toward `plat->low` (no crush, direction -1).
- On `pastDest`: transitions to `waiting`, resets count, plays `sfx_pstop`.

**`waiting` state:**
- Decrements `plat->count` each tic.
- When count reaches zero: if the floor is at `plat->low`, switches to `up`; otherwise switches to `down`. Plays `sfx_pstart`.
- Falls through to `in_stasis` which is a no-op break.

**`in_stasis` state:**
- No action. The platform is paused; `T_PlatRaise` still runs but does nothing. The thinker function pointer is nulled out by `EV_StopPlat` when a platform is put into stasis (see below), but the status is kept for restoration.

---

### `EV_DoPlat`

```c
int EV_DoPlat(line_t *line, plattype_e type, int amount)
```

**Purpose:** Linedef-triggered function that creates and activates platform thinkers for all sectors matching the line's tag. This is the entry point called by `p_spec.c` when a player or projectile activates a platform trigger line.

**Parameters:**
- `line` - the triggering linedef; provides the tag and (for texture-change types) the first sidedef's sector.
- `type` - the platform behaviour type (`plattype_e` from `p_local.h`).
- `amount` - height offset used only by `raiseAndChange`.

**Returns:** `1` if at least one sector was activated, `0` otherwise.

**Key logic:**
1. For `perpetualRaise` type, first calls `P_ActivateInStasis(line->tag)` to wake any stalled perpetual platforms with the same tag before creating new ones.
2. Iterates over all sectors with the matching tag via `P_FindSectorFromLineTag`.
3. Skips any sector that already has `specialdata` set (already has an active special).
4. Allocates a `plat_t` from `PU_LEVSPEC` zone memory and calls `P_AddThinker`.
5. Sets common fields: `type`, `sector`, `sector->specialdata = plat`, thinker callback `T_PlatRaise`, `crush = false`, `tag`.

**Platform type initialisation:**

| Type | Speed | Low | High | Wait | Initial Status | Notes |
|------|-------|-----|------|------|----------------|-------|
| `raiseToNearestAndChange` | `PLATSPEED/2` | - | next higher floor | 0 | `up` | Copies floor texture from trigger line's front sidedef sector. Clears sector special damage. |
| `raiseAndChange` | `PLATSPEED/2` | - | current floor + amount | 0 | `up` | Copies floor texture. |
| `downWaitUpStay` | `PLATSPEED*4` | lowest surrounding floor (capped to current) | current floor | `35*PLATWAIT` | `down` | Classic platform: lowers, waits, comes back. |
| `blazeDWUS` | `PLATSPEED*8` | lowest surrounding floor (capped) | current floor | `35*PLATWAIT` | `down` | Faster version of `downWaitUpStay`. |
| `perpetualRaise` | `PLATSPEED` | lowest surrounding floor (capped) | highest surrounding floor (capped) | `35*PLATWAIT` | random `up` or `down` | Runs forever; start direction is random. |

---

### `P_ActivateInStasis`

```c
void P_ActivateInStasis(int tag)
```

**Purpose:** Reactivates all `in_stasis` platforms whose tag matches the given value. Restores the platform's pre-stasis status and reinstalls the `T_PlatRaise` thinker callback.

**Parameters:**
- `tag` - the sector tag to match against active platforms.

**Key logic:**
- Iterates `activeplats[]`.
- For each non-null entry with matching tag and `status == in_stasis`, sets `status = oldstatus` and restores `thinker.function.acp1 = T_PlatRaise`.

---

### `EV_StopPlat`

```c
void EV_StopPlat(line_t *line)
```

**Purpose:** Linedef trigger that freezes all active platforms with a given tag by putting them into `in_stasis`. The platform remembers its current status in `oldstatus` so it can be resumed later.

**Parameters:**
- `line` - the triggering linedef; `line->tag` identifies which platforms to stop.

**Key logic:**
- Iterates `activeplats[]`.
- For each non-null entry not already in stasis with the matching tag: saves `status` to `oldstatus`, sets `status = in_stasis`, and nulls the thinker function pointer (`acv = NULL`). Nulling the function pointer means the thinker still exists in the list but does nothing each tic.

---

### `P_AddActivePlat`

```c
void P_AddActivePlat(plat_t *plat)
```

**Purpose:** Registers a newly created platform thinker in the `activeplats[]` registry so it can be found by tag-based stop/start operations.

**Parameters:**
- `plat` - the platform to register.

**Key logic:**
- Scans `activeplats[]` for the first NULL slot and assigns it.
- Calls `I_Error` with "no more plats!" if no slot is available (`MAXPLATS` exceeded).

---

### `P_RemoveActivePlat`

```c
void P_RemoveActivePlat(plat_t *plat)
```

**Purpose:** Unregisters a completed or destroyed platform from the `activeplats[]` registry, clears the sector's `specialdata` pointer, and removes its thinker from the thinker list.

**Parameters:**
- `plat` - the platform to remove.

**Key logic:**
- Scans `activeplats[]` to find the matching pointer.
- Clears `sector->specialdata = NULL` so the sector can accept new specials.
- Calls `P_RemoveThinker` to schedule the thinker for deferred free.
- Sets the slot to NULL.
- Calls `I_Error` with "can't find plat!" if the platform was not found (internal consistency check).

---

## Data Structures

`p_plats.c` uses the `plat_t` structure, which is defined in `p_local.h`. Its fields are:

| Type | Field | Purpose |
|------|-------|---------|
| `thinker_t` | `thinker` | Thinker node; must be first for cast compatibility. Callback is `T_PlatRaise`. |
| `sector_t*` | `sector` | The sector whose floor this platform moves. |
| `fixed_t` | `speed` | Movement speed in fixed-point units per tic. |
| `fixed_t` | `low` | Lower height limit for the platform floor. |
| `fixed_t` | `high` | Upper height limit for the platform floor. |
| `int` | `wait` | Number of tics to pause at each extreme before reversing. |
| `int` | `count` | Countdown timer; decremented each tic while `status == waiting`. |
| `plat_e` | `status` | Current FSM state: `up`, `down`, `waiting`, or `in_stasis`. |
| `plat_e` | `oldstatus` | Saved status before stasis; restored by `P_ActivateInStasis`. |
| `boolean` | `crush` | If true, the platform crushes objects when moving upward. Always `false` for platforms created by this file (crush is a ceiling-mover property). |
| `int` | `tag` | Sector tag; used to match platforms to linedef triggers. |
| `plattype_e` | `type` | The platform behaviour variant (see type table above). |

The `plattype_e` enumeration (from `p_local.h`):

| Value | Meaning |
|-------|---------|
| `perpetualRaise` | Oscillates forever between low and high. |
| `downWaitUpStay` | Lowers, waits, raises once, then deactivates. |
| `raiseAndChange` | Raises by a fixed amount and changes floor texture once, then deactivates. |
| `raiseToNearestAndChange` | Raises to the next higher floor and changes texture once, then deactivates. |
| `blazeDWUS` | Fast `downWaitUpStay`. |

The `plat_e` status enumeration:

| Value | Meaning |
|-------|---------|
| `up` | Floor is moving upward toward `high`. |
| `down` | Floor is moving downward toward `low`. |
| `waiting` | Floor is stationary; `count` is counting down. |
| `in_stasis` | Temporarily frozen by a stop trigger. |

---

## Dependencies

| Header / Module | What is used |
|----------------|--------------|
| `i_system.h` | `I_Error` for overflow and corruption checks |
| `z_zone.h` | `Z_Malloc` to allocate `plat_t` from `PU_LEVSPEC` pool |
| `m_random.h` | `P_Random` to randomise `perpetualRaise` initial direction |
| `doomdef.h` | General constants |
| `p_local.h` | `plat_t`, `plattype_e`, `plat_e`, `T_MovePlane`, `P_FindSectorFromLineTag`, `P_FindLowestFloorSurrounding`, `P_FindHighestFloorSurrounding`, `P_FindNextHighestFloor`, `P_AddThinker`, `P_RemoveThinker`, `MAXPLATS`, `PLATSPEED`, `PLATWAIT` |
| `s_sound.h` | `S_StartSound` for movement/stop/start sounds |
| `doomstat.h` | `leveltime`, `sectors[]`, `sides[]` |
| `r_state.h` | `sectors[]`, `sides[]` arrays |
| `sounds.h` | Sound effect IDs: `sfx_stnmov`, `sfx_pstart`, `sfx_pstop` |
