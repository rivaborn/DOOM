# File Overview

`p_telept.c` implements the teleportation mechanic in DOOM. When a player or map object crosses a linedef tagged as a teleporter, this module finds the corresponding teleport destination (an `MT_TELEPORTMAN` map object in a tagged sector), moves the entity to that destination, spawns the teleport fog visual effect at both origin and destination, plays the teleport sound, and briefly freezes the player in place to prevent immediate re-teleportation.

Teleportation is a special line action (triggered by `EV_Teleport`) that is woven into the BSP/map special system. It connects two locations in the map without any apparent spatial path between them, which was one of DOOM's signature gameplay features.

## Global Variables

This file defines no module-level global variables. All variables are local to the single function `EV_Teleport`.

The file does use the following externally-defined globals (from `r_state.h` and `p_local.h`):

| External Variable | Type | Purpose |
|---|---|---|
| `thinkercap` | `thinker_t` | Head/tail sentinel node of the thinker linked list, used to iterate all active thinkers to find teleport destination objects |
| `sectors` | `sector_t*` | Array of all map sectors, indexed to match tagged sectors |
| `numsectors` | `int` | Total number of sectors, bounds the sector search loop |
| `finecosine[]` | `fixed_t[]` | Lookup table for cosine values used to offset fog spawn position |
| `finesine[]` | `fixed_t[]` | Lookup table for sine values used to offset fog spawn position |

## Functions

### `EV_Teleport`

```c
int EV_Teleport(line_t* line, int side, mobj_t* thing)
```

**Purpose:** Executes a teleportation event for a map object (`thing`) that has crossed the triggering linedef (`line`). Finds a teleport destination in the tagged sector, moves the object there, and produces the visual/audio effects.

**Parameters:**
- `line` (`line_t*`): The linedef that was crossed to trigger the teleport. Its `tag` field is used to find the destination sector.
- `side` (`int`): Which side of the line was crossed. `0` = front side (triggers teleport), `1` = back side (suppressed, so the player can exit the destination pad without re-teleporting).
- `thing` (`mobj_t*`): The map object attempting to teleport (player, monster, etc.).

**Return Value:** `int` — Returns `1` on successful teleportation, `0` if teleportation was blocked or failed (missile, back-side crossing, no destination found, `P_TeleportMove` failed).

**Key Logic:**

1. **Missile rejection:** If `thing->flags & MF_MISSILE`, return 0 immediately. Projectiles do not teleport.
2. **Back-side rejection:** If `side == 1`, return 0. This prevents re-teleportation immediately after arriving.
3. **Sector search:** Iterates over all sectors (`numsectors`) looking for any sector whose `tag` matches the linedef's `tag`.
4. **Destination search:** For each matching sector, iterates the global thinker list (`thinkercap`) looking for a thinker whose action function is `P_MobjThinker` (i.e., a live map object) and whose type is `MT_TELEPORTMAN` and whose subsector belongs to the matching sector. This is the teleport destination marker.
5. **Movement:** Calls `P_TeleportMove(thing, m->x, m->y)` to actually move the object. This function handles collision, unlike `P_UnsetThingPosition`/`P_SetThingPosition`. If the move fails, returns 0.
6. **Height adjustment:** Sets `thing->z` to `thing->floorz`. Updates player's `viewz` if applicable.
7. **Fog effects:** Spawns `MT_TFOG` at the old position and at a position slightly in front of the destination marker (offset by 20 units using `finecosine`/`finesine` of the destination angle). Plays `sfx_telept` on each fog object.
8. **Freeze:** Sets `thing->reactiontime = 18` for players, preventing movement for about half a second.
9. **Angle/momentum reset:** Sets `thing->angle` to match the destination marker's angle, and zeroes all momentum (`momx`, `momy`, `momz`).

## Data Structures

This file does not define any new data structures. It operates on the pre-existing engine types:

- `mobj_t` (map object / thing)
- `thinker_t` (thinker linked list node)
- `sector_t` (map sector)
- `line_t` (linedef)

## Dependencies

| File | Reason |
|---|---|
| `doomdef.h` | Core type definitions (`fixed_t`, `angle_t`, boolean, etc.) |
| `s_sound.h` | `S_StartSound()` for playing the teleport sound effect |
| `p_local.h` | `P_TeleportMove()`, `P_SpawnMobj()`, `P_MobjThinker`, `ANGLETOFINESHIFT`, and other play-module internals |
| `sounds.h` | `sfx_telept` sound enumeration constant |
| `r_state.h` | `sectors[]`, `numsectors`, `thinkercap` (renderer state used by play) |
