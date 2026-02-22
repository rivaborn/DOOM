# File Overview

**Source file:** `linuxdoom-1.10/p_mobj.c`

`p_mobj.c` is the heart of DOOM's game-object system. It manages the complete lifecycle of every **map object** (mobj) in the game world: spawning objects from map data or from gameplay events, running their per-tic logic through the thinker system, applying physics (gravity, friction, momentum), and removing objects when they are no longer needed. It also houses the item-respawn queue used in deathmatch and the nightmare-mode monster respawn logic.

Every entity in the world that the simulation cares about — players, monsters, projectiles, items, decorations, effects — is an `mobj_t`. This file owns:

- The **thinker callback** (`P_MobjThinker`) that drives every mobj every game tic.
- The **state machine driver** (`P_SetMobjState`) that advances sprite animation frames and fires action callbacks.
- The **physics routines** (`P_XYMovement`, `P_ZMovement`) that move objects and handle collision response.
- The **spawn and removal** API used throughout the engine.
- **Game-play spawn helpers** for puffs, blood, and missiles.

---

## Global Variables

| Type | Name | Purpose |
|------|------|---------|
| `int` | `test` | Unused debug/scratch variable left in the released source. Never read or written after declaration. |
| `mapthing_t[ITEMQUESIZE]` | `itemrespawnque` | Circular queue of map-thing spawn records for items that have been removed in deathmatch mode 2 and are waiting to respawn. Each entry is a copy of the original `mapthing_t` from the map lump. |
| `int[ITEMQUESIZE]` | `itemrespawntime` | Parallel array to `itemrespawnque`. Stores the `leveltime` value at which each item was removed, so the 30-second delay before respawn can be enforced. |
| `int` | `iquehead` | Write pointer (head) into the item-respawn circular queue. New removals are recorded here, then the index is incremented modulo `ITEMQUESIZE`. |
| `int` | `iquetail` | Read pointer (tail) into the item-respawn circular queue. The oldest pending respawn is at this index. When `iquehead == iquetail`, the queue is empty. |
| `fixed_t` | `attackrange` | Declared `extern` in this file; defined in `p_map.c`. Used by `P_SpawnPuff` to suppress sparks when a melee-range attack hits a wall. |

---

## Functions

### `P_SetMobjState`

```c
boolean P_SetMobjState(mobj_t *mobj, statenum_t state)
```

**Purpose:** Drives the mobj's finite-state machine (FSM). Sets the object to the given state and immediately runs any action function attached to that state. If the new state has zero tics it loops, advancing through zero-tic states until a state with a non-zero tic count is found, or until `S_NULL` is reached.

**Parameters:**
- `mobj` - the object whose state is being changed.
- `state` - target state number from the `statenum_t` enum (defined in `info.h`).

**Returns:** `true` if the mobj is still alive after the transition; `false` if the state was `S_NULL`, which triggers `P_RemoveMobj` and frees the object.

**Key logic:**
1. If `state == S_NULL`, call `P_RemoveMobj` and return `false`.
2. Look up `states[state]`, copy `tics`, `sprite`, and `frame` into the mobj.
3. If the state has an `action.acp1` callback, call it with the mobj.
4. Read `st->nextstate` and loop if `mobj->tics == 0` (zero-tic transitional states).

---

### `P_ExplodeMissile`

```c
void P_ExplodeMissile(mobj_t *mo)
```

**Purpose:** Triggers the death/explosion sequence of a missile object. Zeroes all momentum, transitions to the mobj's `deathstate`, randomises the initial tic count slightly (so multiple explosions do not all flash in sync), clears the `MF_MISSILE` flag, and plays the death sound.

**Parameters:**
- `mo` - the missile mobj to explode.

**Key logic:**
- Momentum cleared to prevent continued movement during the explosion animation.
- `mo->tics -= P_Random() & 3` introduces up to 3 tic jitter; clamped to a minimum of 1.
- Clearing `MF_MISSILE` means the explosion sprite is treated as a regular object for collision purposes.

---

### `P_XYMovement`

```c
void P_XYMovement(mobj_t *mo)
```

**Purpose:** Applies horizontal momentum to an mobj for one game tic. Handles the full collision pipeline: movement capping, step-subdivision (moves larger than `MAXMOVE/2` are split in half iteratively), collision response (slide for the player, explosion for missiles, momentum zeroing for others), sky-missile removal, and friction/deceleration when on the floor.

**Parameters:**
- `mo` - the mobj to move.

**Constants used:**
- `STOPSPEED` (`0x1000`) - threshold below which momentum is snapped to zero.
- `FRICTION` (`0xE800`) - fixed-point multiplier (~0.906) applied every tic to slow grounded objects.
- `MAXMOVE` - maximum allowed momentum per tic (defined in `p_local.h`).

**Key logic:**
1. Early-out if both `momx` and `momy` are zero, with special `MF_SKULLFLY` reset.
2. Cap momentum to `MAXMOVE`.
3. Loop: if remaining move exceeds `MAXMOVE/2`, advance half the distance; otherwise advance all. Call `P_TryMove`.
4. On blocked move: players slide (`P_SlideMove`); missiles against a sky ceiling are silently removed; other missiles explode; non-player non-missiles stop.
5. After movement: missiles and flying objects skip friction. Corpses on steps may keep sliding.
6. Apply `FRICTION` or zero momentum when below `STOPSPEED` and the player is not pressing movement keys.
7. Snap the player back to the `S_PLAY` idle state if they were in a walk animation.

---

### `P_ZMovement`

```c
void P_ZMovement(mobj_t *mo)
```

**Purpose:** Applies vertical momentum for one game tic. Handles smooth step-up view adjustment for players, floating-enemy height tracking, floor and ceiling collision, gravity application, and hard-landing squat effect.

**Parameters:**
- `mo` - the mobj to move vertically.

**Key logic:**
1. **Smooth step-up:** If a player's z is below `floorz`, reduce `viewheight` proportionally and set `deltaviewheight` to recover smoothly.
2. Add `momz` to `mo->z`.
3. **MF_FLOAT enemies:** If the object has a target and is a floater (but not `MF_SKULLFLY` or `MF_INFLOAT`), adjust z toward the target's midpoint by `FLOATSPEED`.
4. **Floor collision:** Zero `momz`; for players falling faster than `GRAVITY*8`, play `sfx_oof` and apply a squash to `deltaviewheight`. Missiles without `MF_NOCLIP` explode. The Lost Soul (`MF_SKULLFLY`) bounces its z-momentum.
5. **Gravity:** If not floor-bound and not `MF_NOGRAVITY`, apply `GRAVITY` each tic (first tic without floor contact applies `GRAVITY*2`).
6. **Ceiling collision:** Cap z to `ceilingz - height`, zero `momz` (unless `MF_SKULLFLY` bounces), explode missiles.

---

### `P_NightmareRespawn`

```c
void P_NightmareRespawn(mobj_t *mobj)
```

**Purpose:** Respawns a dead monster at its original map spawn point in Nightmare difficulty. Teleport fog effects are spawned at both the old and new locations.

**Parameters:**
- `mobj` - the dead monster mobj (tics == -1, meaning it is in a permanent state).

**Key logic:**
1. Convert spawn-point coordinates to fixed-point.
2. Use `P_CheckPosition` to confirm nothing is blocking the destination.
3. Spawn `MT_TFOG` at the old corpse location and at the new spawn point; play `sfx_telept` at each.
4. Create a new mobj of the same type at the spawn point, copying `spawnpoint`, `angle`, and the `MF_AMBUSH` flag.
5. Set `reactiontime = 18` (about half a second) so the monster does not immediately attack.
6. Remove the old corpse with `P_RemoveMobj`.

---

### `P_MobjThinker`

```c
void P_MobjThinker(mobj_t *mobj)
```

**Purpose:** Per-tic update callback for every mobj in the world. This is the function registered as each mobj's `thinker.function`. It is called by the thinker list runner in `p_tick.c` once per game tic per live mobj.

**Parameters:**
- `mobj` - the mobj being ticked.

**Key logic:**
1. **Horizontal movement:** If `momx`, `momy`, or `MF_SKULLFLY`, call `P_XYMovement`. After the call, check whether the mobj was deleted (function pointer sentinel `-1`).
2. **Vertical movement:** If not at floor height or has `momz`, call `P_ZMovement`. Same deletion check.
3. **State machine:** If `tics != -1`, decrement the tic counter. When it reaches zero, call `P_SetMobjState` to advance to the next animation state.
4. **Infinite state / nightmare respawn:** If `tics == -1` (permanent state, typically a dead monster), check `MF_COUNTKILL` and `respawnmonsters`. After 12*35 tics (12 seconds), with a random probability check (`P_Random() <= 4`) throttled to once every 32 tics, call `P_NightmareRespawn`.

**Critical sentinel:** The thinker deletion check `mobj->thinker.function.acv == (actionf_v)(-1)` is an acknowledged hack. `P_RemoveThinker` sets the function pointer to `-1` as a deferred-free marker. The FIXME comment in the source acknowledges this should be replaced with a proper NULL/NOP sentinel.

---

### `P_SpawnMobj`

```c
mobj_t *P_SpawnMobj(fixed_t x, fixed_t y, fixed_t z, mobjtype_t type)
```

**Purpose:** The primary factory function for all map objects. Allocates an `mobj_t` from the zone allocator, initialises all fields from the `mobjinfo` table, places the object in the world (sector and blockmap links), and registers it as an active thinker.

**Parameters:**
- `x`, `y` - world-space position in fixed-point units.
- `z` - vertical position; may be the special values `ONFLOORZ` or `ONCEILINGZ`.
- `type` - `mobjtype_t` index into the `mobjinfo[]` table.

**Returns:** Pointer to the newly created `mobj_t`.

**Key logic:**
1. `Z_Malloc(sizeof(*mobj), PU_LEVEL, NULL)` — allocated from the level memory pool; freed automatically at level unload.
2. All fields are zero-initialised with `memset`, then populated from `mobjinfo[type]`.
3. Reaction time is only set on non-Nightmare skill levels; Nightmare monsters attack immediately.
4. `lastlook` is randomised so monsters do not all start looking for the same player.
5. Initial state is set directly (not via `P_SetMobjState`) because action callbacks cannot safely fire during spawn.
6. `P_SetThingPosition` links the mobj into its subsector and blockmap cell.
7. `z == ONFLOORZ` places the object at `floorheight`; `z == ONCEILINGZ` places it at `ceilingheight - height`.
8. `P_MobjThinker` is registered as the thinker callback, and `P_AddThinker` inserts it into the global thinker list.

---

### `P_RemoveMobj`

```c
void P_RemoveMobj(mobj_t *mobj)
```

**Purpose:** Removes an mobj from the world. Handles the deathmatch item-respawn queue, unlinks the object from the sector and blockmap spatial structures, stops any sound it was emitting, and schedules it for deferred memory release via `P_RemoveThinker`.

**Parameters:**
- `mobj` - the object to remove.

**Key logic:**
1. If the mobj is a special pickup (`MF_SPECIAL`), was not dropped by a monster (`!MF_DROPPED`), and is not the invulnerability or invisibility sphere, its `spawnpoint` is pushed onto the circular `itemrespawnque` with the current `leveltime`. If the queue is full, the oldest entry is overwritten.
2. `P_UnsetThingPosition` unlinks from sector and blockmap lists.
3. `S_StopSound` halts any positional audio.
4. `P_RemoveThinker` marks the thinker for deferred removal (sets function pointer to `-1`).

---

### `P_RespawnSpecials`

```c
void P_RespawnSpecials(void)
```

**Purpose:** Called once per tic during deathmatch mode 2 games. Checks the item-respawn queue and respawns the oldest item if it has been waiting at least 30 seconds (30*35 tics).

**Key logic:**
1. Returns immediately unless `deathmatch == 2`.
2. If queue is empty (`iquehead == iquetail`), returns.
3. Checks `leveltime - itemrespawntime[iquetail] < 30*35`; if not enough time has passed, returns.
4. Spawns `MT_IFOG` at the respawn location with `sfx_itmbk`.
5. Searches `mobjinfo[]` for the mobj type matching `mthing->type` (the editor number).
6. Spawns the item with a fresh mobj; restores `spawnpoint` and `angle`.
7. Advances `iquetail` to consume the entry.

---

### `P_SpawnPlayer`

```c
void P_SpawnPlayer(mapthing_t *mthing)
```

**Purpose:** Spawns the player entity at a player-start map thing. Handles both initial spawn and respawn from `PST_REBORN` state. Initialises all player HUD and view state, sets up weapon sprites, and activates the status bar and heads-up display for the console player.

**Parameters:**
- `mthing` - the map thing record for a player start (type 1-4).

**Key logic:**
1. Returns immediately if `playeringame[mthing->type-1]` is false.
2. If the player state is `PST_REBORN`, calls `G_PlayerReborn` to reset inventory.
3. Spawns `MT_PLAYER` via `P_SpawnMobj`.
4. For players 2-4 (multiplayer), sets translation flags in `mobj->flags` for colour remapping.
5. Resets `damagecount`, `bonuscount`, `extralight`, `fixedcolormap`, `viewheight`.
6. In deathmatch, grants all keycards.
7. Calls `P_SetupPsprites` to start weapon bring-up.
8. Activates `ST_Start` and `HU_Start` for the console player.

---

### `P_SpawnMapThing`

```c
void P_SpawnMapThing(mapthing_t *mthing)
```

**Purpose:** Spawns a single map thing from the WAD THINGS lump during level loading. Handles player starts, deathmatch starts, skill-level filtering, the `-nomonsters` command-line flag, and deathmatch exclusions.

**Parameters:**
- `mthing` - a `mapthing_t` record read from the map's THINGS lump (already byte-swapped to host order).

**Key logic:**
1. Type 11 is a deathmatch start; stored in `deathmatchstarts[]` array (up to 10 entries).
2. Types 1-4 are player starts; saved in `playerstarts[]`. In non-deathmatch mode, immediately calls `P_SpawnPlayer`.
3. Skill-level filtering: `options` bits 1, 2, 4 correspond to Easy, Medium, and Hard/Nightmare respectively. The `baby` skill uses bit 1; `nightmare` uses bit 4. The multiplayer-only bit (16) is checked.
4. Looks up mobj type by editor number (`doomednum`) in `mobjinfo[]`; calls `I_Error` if not found.
5. Items with `MF_NOTDMATCH` are skipped in deathmatch.
6. Monsters are skipped when `-nomonsters` is active.
7. Randomises initial tic count slightly (`1 + P_Random() % tics`) to desynchronise animations.
8. Increments `totalkills` and `totalitems` as appropriate for the intermission tally.

---

### `P_SpawnPuff`

```c
void P_SpawnPuff(fixed_t x, fixed_t y, fixed_t z)
```

**Purpose:** Spawns a bullet-hit puff effect at a world position. Used when a hitscan attack hits a wall or non-bleeding target. Adds vertical randomness and slight upward momentum. For melee-range attacks, starts the puff at frame `S_PUFF3` to skip the spark frames.

**Parameters:**
- `x`, `y`, `z` - world position of the impact.

---

### `P_SpawnBlood`

```c
void P_SpawnBlood(fixed_t x, fixed_t y, fixed_t z, int damage)
```

**Purpose:** Spawns a blood splat effect when a bleeding target is hit. The initial state of the blood sprite depends on damage amount: high damage (>12) uses the full three-frame blood spray; medium damage (9-12) skips the first frame; low damage (<9) skips two frames, producing a smaller splat.

**Parameters:**
- `x`, `y`, `z` - world position of the hit.
- `damage` - hit-point damage dealt; controls which blood animation frame is used.

---

### `P_CheckMissileSpawn`

```c
void P_CheckMissileSpawn(mobj_t *th)
```

**Purpose:** Finalises missile creation by nudging the missile forward half a step and checking whether it immediately hits something. If it does, it explodes immediately. This prevents missiles from spawning inside the shooter and detonating on them.

**Parameters:**
- `th` - the freshly spawned missile mobj.

**Key logic:**
- Advances `x`, `y`, `z` by half the momentum vector.
- Randomises tic count by up to 3 tics for animation desynchronisation.
- Calls `P_TryMove`; if blocked, calls `P_ExplodeMissile`.

---

### `P_SpawnMissile`

```c
mobj_t *P_SpawnMissile(mobj_t *source, mobj_t *dest, mobjtype_t type)
```

**Purpose:** Spawns a monster-fired missile aimed at a target mobj. Used for monster projectile attacks (e.g., imp fireballs, cacodemon balls). Applies a random angular offset if the target has the `MF_SHADOW` (spectre) flag to simulate the partial invisibility effect.

**Parameters:**
- `source` - the mobj firing the missile (sets `th->target`).
- `dest` - the target mobj to aim at.
- `type` - the mobj type of the projectile.

**Returns:** Pointer to the spawned missile mobj.

**Key logic:**
1. Spawns at `source->z + 32` (4*8*FRACUNIT).
2. Calculates aim angle with `R_PointToAngle2`; adds random jitter if target is shadowed.
3. Sets `momx` and `momy` from `speed * cos(angle)` and `speed * sin(angle)`.
4. Calculates `momz` from the height difference divided by approximate travel time.
5. Calls `P_CheckMissileSpawn` to handle immediate-wall scenarios.

---

### `P_SpawnPlayerMissile`

```c
void P_SpawnPlayerMissile(mobj_t *source, mobjtype_t type)
```

**Purpose:** Spawns a player-fired missile with auto-aim. Attempts three `P_AimLineAttack` sweeps (straight, +1<<26, -2<<26) to find a nearby monster within the autoaim cone. If no target is found, fires straight ahead with zero vertical slope.

**Parameters:**
- `source` - the player's mobj.
- `type` - the mobj type to spawn (e.g., `MT_ROCKET`, `MT_PLASMA`, `MT_BFG`).

**Key logic:**
- Three-pass aim sweep mimics DOOM's auto-aim behaviour.
- Missile spawned at player's z + 32 units.
- `momz = speed * slope` where `slope` comes from `P_AimLineAttack`.

---

## Data Structures

No new data structures are defined in `p_mobj.c`. The file uses `mobj_t` (from `p_mobj.h`), `mapthing_t` (from `doomdata.h`), `state_t` and `mobjinfo_t` (from `info.h`), and `player_t` (from `d_player.h`).

The item-respawn queue is a pair of parallel arrays forming a power-of-two circular buffer:

```
itemrespawnque[ITEMQUESIZE]   -- mapthing_t records
itemrespawntime[ITEMQUESIZE]  -- leveltime stamps
iquehead                      -- next write position
iquetail                      -- next read position
```

The buffer wraps using bitwise AND: `(index + 1) & (ITEMQUESIZE - 1)`. `ITEMQUESIZE` must therefore be a power of two (it is 128, defined in `p_local.h`).

---

## Dependencies

| Header / Module | What is used |
|----------------|--------------|
| `i_system.h` | `I_Error` for fatal errors |
| `z_zone.h` | `Z_Malloc` for mobj allocation |
| `m_random.h` | `P_Random` for randomness in tic jitter, respawn, aim |
| `doomdef.h` | Game-wide constants (`GRAVITY`, `FRACUNIT`, etc.) |
| `p_local.h` | `P_TryMove`, `P_SlideMove`, `P_AimLineAttack`, `P_LineAttack`, `P_CheckPosition`, `P_SetThingPosition`, `P_UnsetThingPosition`, `P_AddThinker`, `P_RemoveThinker`, `P_FindSectorFromLineTag`, constants `MAXMOVE`, `FLOATSPEED`, `ITEMQUESIZE`, `PLATWAIT`, etc. |
| `sounds.h` | Sound effect IDs (`sfx_oof`, `sfx_telept`, `sfx_itmbk`) |
| `st_stuff.h` | `ST_Start` to wake the status bar |
| `hu_stuff.h` | `HU_Start` to wake the heads-up display |
| `s_sound.h` | `S_StartSound`, `S_StopSound` |
| `doomstat.h` | Global state: `gameskill`, `deathmatch`, `respawnmonsters`, `nomonsters`, `leveltime`, `totalkills`, `totalitems`, `players[]`, `playeringame[]`, `consoleplayer`, `deathmatchstarts[]`, `deathmatch_p`, `playerstarts[]`, `linetarget` |
| `info.h` | `states[]`, `mobjinfo[]`, `statenum_t`, `mobjtype_t` |
| `r_main.h` | `R_PointInSubsector`, `R_PointToAngle2` |
| `tables.h` | `finecosine[]`, `finesine[]`, `ANGLETOFINESHIFT` |
| `g_game.h` | `G_PlayerReborn` (extern declaration in this file) |
| `p_pspr.h` | `P_SetupPsprites` |
