# File Overview

`p_tick.c` is the core game-logic ticker for DOOM. It owns the **thinker system** — a doubly-linked list of `thinker_t` objects that represent any entity in the game world that needs to perform periodic computation (doors, lifts, monsters, projectiles, player physics, etc.). Every game tic, `P_Ticker` drives the entire play simulation: it runs all thinkers, processes special sector effects, handles respawning, and increments the level timer.

The file's comment header mentions "Archiving: SaveGame I/O" alongside "Thinker, Ticker", which reflects the historical fact that the thinker list and level state were the primary subjects of savegame serialization.

## Global Variables

| Type | Name | Description |
|---|---|---|
| `int` | `leveltime` | Counts game tics elapsed since the current level started. Used for par-time comparisons, timed specials, and sector animations. Incremented once per `P_Ticker` call. |
| `thinker_t` | `thinkercap` | The sentinel node for the doubly-linked circular thinker list. Both `thinkercap.prev` and `thinkercap.next` point into the list; the list is considered empty when both point back to `&thinkercap`. All active thinkers live between `thinkercap.next` and `thinkercap.prev`. |

## Functions

### `P_InitThinkers`

```c
void P_InitThinkers(void)
```

**Purpose:** Initializes the thinker list to the empty state by making `thinkercap` point to itself in both directions.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:** Sets `thinkercap.prev = thinkercap.next = &thinkercap`. This is the standard sentinel/circular-list initialization pattern. Called once at level load.

---

### `P_AddThinker`

```c
void P_AddThinker(thinker_t* thinker)
```

**Purpose:** Appends a new thinker to the **end** of the thinker list (just before the sentinel `thinkercap`).

**Parameters:**
- `thinker` (`thinker_t*`): Pointer to the thinker structure to add. The caller is responsible for allocating it (typically via `Z_Malloc`). The thinker's `function` field should already be set before calling this.

**Return Value:** `void`

**Key Logic:** Standard doubly-linked list insertion at the tail:
1. The current last element (`thinkercap.prev`) links its `next` to `thinker`.
2. `thinker->next` is set to `&thinkercap`.
3. `thinker->prev` is set to the old tail.
4. `thinkercap.prev` is updated to point to `thinker`.

---

### `P_RemoveThinker`

```c
void P_RemoveThinker(thinker_t* thinker)
```

**Purpose:** Marks a thinker for removal from the list. The actual deallocation is **deferred** until the thinker's next turn in `P_RunThinkers`.

**Parameters:**
- `thinker` (`thinker_t*`): The thinker to remove.

**Return Value:** `void`

**Key Logic:** Rather than immediately removing from the list and freeing memory (which would be unsafe during iteration), this function sets `thinker->function.acv = (actionf_v)(-1)`. The sentinel value `-1` is recognized by `P_RunThinkers` as the "remove me" signal. The comment marks this as a NOP / lazy deallocation scheme.

---

### `P_AllocateThinker`

```c
void P_AllocateThinker(thinker_t* thinker)
```

**Purpose:** Stub function; body is empty. Was presumably intended to handle thinker memory allocation but was never implemented — callers use `Z_Malloc` directly.

**Parameters:**
- `thinker` (`thinker_t*`): Unused.

**Return Value:** `void`

---

### `P_RunThinkers`

```c
void P_RunThinkers(void)
```

**Purpose:** Iterates the entire thinker list and either removes dead thinkers or calls each live thinker's action function.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:**

1. Starts at `thinkercap.next` and walks the list until returning to `&thinkercap`.
2. For each thinker, checks if `function.acv == (actionf_v)(-1)`:
   - If true (marked for removal): splices the node out of the list and calls `Z_Free(currentthinker)` to return the memory to the zone allocator. Iteration continues with the next node, which is now `currentthinker->next` (captured before the free).
   - If false and `function.acp1` is non-null: calls `currentthinker->function.acp1(currentthinker)`, which dispatches to the appropriate think function (e.g., `P_MobjThinker`, `T_MoveCeiling`, etc.).
3. Each thinker runs exactly once per call.

---

### `P_Ticker`

```c
void P_Ticker(void)
```

**Purpose:** The top-level per-tic game logic driver. Called once per game tic (35 times per second). Orchestrates all gameplay simulation for one frame.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:**

1. **Pause check:** If `paused` is true, returns immediately.
2. **Menu pause:** If not in a net game, the menu is active, no demo is playing, and the view has been initialized (indicated by `players[consoleplayer].viewz != 1`), returns immediately. This lets the menu overlay without advancing game state.
3. **Player think:** For each player slot (`0..MAXPLAYERS-1`), if `playeringame[i]` is set, calls `P_PlayerThink(&players[i])` to process movement, weapon, and power-up logic for that player.
4. **Run thinkers:** Calls `P_RunThinkers()` to execute all active sector/object thinkers.
5. **Update specials:** Calls `P_UpdateSpecials()` to animate scrolling textures, flashing lights, etc.
6. **Respawn specials:** Calls `P_RespawnSpecials()` to handle item respawning in Nightmare mode.
7. **Level timer:** Increments `leveltime`.

## Data Structures

This file does not define new data structures. It uses `thinker_t` from `p_local.h`/`doomdata.h`:

```c
// thinker_t is the base struct that all thinking objects share as their first field.
// The function union allows a single pointer-sized slot to hold different function pointer types.
typedef struct thinker_s {
    struct thinker_s*   prev;
    struct thinker_s*   next;
    actionf_t           function;
} thinker_t;
```

## Dependencies

| File | Reason |
|---|---|
| `z_zone.h` | `Z_Free()` for deallocating removed thinkers |
| `p_local.h` | `thinker_t` definition, `P_PlayerThink`, `P_UpdateSpecials`, `P_RespawnSpecials`, `MAXPLAYERS`, and related constants |
| `doomstat.h` | `paused`, `netgame`, `menuactive`, `demoplayback`, `players[]`, `playeringame[]`, `consoleplayer` global game state variables |
