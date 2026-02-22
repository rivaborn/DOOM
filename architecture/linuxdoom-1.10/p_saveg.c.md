# File Overview

`p_saveg.c` implements DOOM's save game archiving and restoration system. It is responsible for serializing and deserializing the complete game state to and from a raw byte buffer (`save_p`), which is later written to disk by higher-level game code (`g_game.c`). The serialization approach is a direct binary memory dump of live game structures, with pointer values converted to numeric indices (swizzling) on save and converted back to valid pointers on load (unswizzling).

The file is organized into four paired archive/unarchive function groups:
1. **Players** - Player state including inventory, powers, and psprite state indices.
2. **World** - The mutable parts of the map geometry: sector heights, lighting, textures, and line flags/specials.
3. **Thinkers** - Mobile objects (map objects / monsters / items).
4. **Specials** - Active sector-effect thinkers: ceilings, doors, floors, platforms, light effects.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `byte*` | `save_p` | The current write/read cursor into the save game buffer. All archive functions advance this pointer as they write data; all unarchive functions advance it as they read data. Declared `extern` in `p_saveg.h`. |

## Macros

### `PADSAVEP()`
```c
#define PADSAVEP() save_p += (4 - ((int) save_p & 3)) & 3
```
Aligns `save_p` to a 4-byte boundary before writing or reading a structure. This is required for correct behavior on architectures (such as SGI MIPS and Gecko/PowerPC) that require aligned memory accesses for multi-byte types. Without this, reading an `int` or `fixed_t` from an unaligned address would produce corrupt data or a bus error.

## Data Structures

### `thinkerclass_t` (local enum)
```c
typedef enum { tc_end, tc_mobj } thinkerclass_t;
```
Tag byte written before each archived mobj, and as a list terminator (`tc_end`) at the end of the thinker list.

### `specials_e` (local enum)
```c
enum { tc_ceiling, tc_door, tc_floor, tc_plat, tc_flash, tc_strobe, tc_glow, tc_endspecials } specials_e;
```
Tag byte written before each archived special thinker. `tc_endspecials` is the list terminator.

## Functions

### `P_ArchivePlayers`
```c
void P_ArchivePlayers(void)
```
**Purpose:** Serializes all active players' `player_t` structures into the save buffer.

**Parameters:** None.

**Return value:** None (writes into `save_p`).

**Key logic:**
- Iterates over `MAXPLAYERS` slots; skips slots where `playeringame[i]` is false.
- Calls `PADSAVEP()` for alignment before each player.
- Copies the entire `player_t` struct via `memcpy`.
- Converts `psprites[j].state` pointer to an integer offset from the `states` array base. This is necessary because raw pointers are not valid after a reload; they must be reconstructed from indices.

### `P_UnArchivePlayers`
```c
void P_UnArchivePlayers(void)
```
**Purpose:** Restores all active players from the save buffer.

**Parameters:** None.

**Return value:** None (reads from `save_p`).

**Key logic:**
- Mirrors `P_ArchivePlayers`: iterates active players, aligns, and `memcpy`s.
- Sets `players[i].mo`, `players[i].message`, and `players[i].attacker` to `NULL` because these pointers are re-established by `P_UnArchiveThinkers` when the mobj list is restored.
- Converts `psprites[j].state` integer offset back to a `state_t*` pointer by indexing into the `states` array.

### `P_ArchiveWorld`
```c
void P_ArchiveWorld(void)
```
**Purpose:** Serializes the mutable state of all sectors and line sides.

**Parameters:** None.

**Return value:** None (writes into `save_p` via local `put` pointer).

**Key logic:**
- Uses a local `short* put` pointer (cast from `save_p`) for efficient short-word writes.
- **Sectors:** For each sector, writes: `floorheight >> FRACBITS`, `ceilingheight >> FRACBITS`, `floorpic`, `ceilingpic`, `lightlevel`, `special`, `tag`. Heights are saved in map units (integer), not fixed-point fractions.
- **Lines/Sides:** For each linedef, writes `flags`, `special`, `tag`. Then for each of its sides (0 and 1), writes: `textureoffset >> FRACBITS`, `rowoffset >> FRACBITS`, `toptexture`, `bottomtexture`, `midtexture`. Skips sides with `sidenum[j] == -1` (one-sided lines).
- Updates `save_p` from the local `put` pointer at the end.

### `P_UnArchiveWorld`
```c
void P_UnArchiveWorld(void)
```
**Purpose:** Restores sector and line side state from the save buffer. Exact mirror of `P_ArchiveWorld`.

**Parameters:** None.

**Return value:** None.

**Key logic:**
- Restores all archived fields, shifting heights back up by `FRACBITS`.
- Resets `sector->specialdata` and `sector->soundtarget` to `0` since these are live pointers that will be re-established by `P_UnArchiveSpecials` and sound propagation.

### `P_ArchiveThinkers`
```c
void P_ArchiveThinkers(void)
```
**Purpose:** Serializes all active map objects (mobjs) from the thinker list.

**Parameters:** None.

**Return value:** None.

**Key logic:**
- Iterates the global doubly-linked thinker list (`thinkercap.next` to `thinkercap`).
- Only serializes thinkers whose function is `P_MobjThinker` (i.e., living map objects). All other thinker types (sector effects) are silently skipped here — they are handled by `P_ArchiveSpecials`.
- Writes tag byte `tc_mobj`, aligns, copies the `mobj_t`.
- Swizzles `mobj->state` pointer to an index into `states[]`.
- Swizzles `mobj->player` pointer to a 1-based player index (0 means no player owner; `(player - players) + 1` stores the index + 1 to distinguish from NULL).
- Writes `tc_end` as a list terminator.

### `P_UnArchiveThinkers`
```c
void P_UnArchiveThinkers(void)
```
**Purpose:** Restores mobj thinkers from the save buffer.

**Parameters:** None.

**Return value:** None.

**Key logic:**
- First removes all existing thinkers: for each `P_MobjThinker` thinker calls `P_RemoveMobj`; for others calls `Z_Free`. Then calls `P_InitThinkers()` to reset the list.
- Reads tag bytes in a loop until `tc_end`.
- For `tc_mobj`: allocates a new `mobj_t` via `Z_Malloc`, reads data, unswizzles `state` and `player`, then calls `P_SetThingPosition`, restores `mobj->info` and floor/ceiling z values, sets `thinker.function.acp1 = P_MobjThinker`, and adds to the thinker list.

### `P_ArchiveSpecials`
```c
void P_ArchiveSpecials(void)
```
**Purpose:** Serializes all active sector-effect thinkers (ceilings, doors, floors, platforms, light effects).

**Parameters:** None.

**Return value:** None.

**Key logic:**
- Iterates the thinker list, identifies each special thinker by its `function.acp1` value.
- Handles the special case of paused ceilings: when a ceiling is in stasis, its `function.acv` is `NULL`, so it cannot be identified by function pointer alone. Instead, the code searches `activeceilings[]` to determine if a NULL-function thinker is an active ceiling.
- For each recognized special type, writes the type tag byte, aligns, copies the struct, and swizzles the embedded `sector*` pointer to an index (`sector - sectors`).
- Supported types: `ceiling_t` (`tc_ceiling`), `vldoor_t` (`tc_door`), `floormove_t` (`tc_floor`), `plat_t` (`tc_plat`), `lightflash_t` (`tc_flash`), `strobe_t` (`tc_strobe`), `glow_t` (`tc_glow`).
- Terminates with `tc_endspecials`.

### `P_UnArchiveSpecials`
```c
void P_UnArchiveSpecials(void)
```
**Purpose:** Restores all sector-effect thinkers from the save buffer.

**Parameters:** None.

**Return value:** None.

**Key logic:**
- Reads tag bytes in a loop until `tc_endspecials`.
- For each type: allocates via `Z_Malloc`, reads data, unswizzles sector pointer, restores sector's `specialdata` pointer, restores function pointer, and calls `P_AddThinker`.
- For `tc_ceiling` and `tc_plat`, also calls `P_AddActiveCeiling` / `P_AddActivePlat` (which manage the active lists) only if the thinker function was not NULL (i.e., was not in stasis at save time).
- Calls `I_Error` on any unrecognized tag byte to catch corrupted saves.

## Dependencies

| File | Reason |
|------|--------|
| `i_system.h` | `I_Error` for error reporting |
| `z_zone.h` | `Z_Malloc`, `Z_Free` for thinker allocation |
| `p_local.h` | Map object definitions, thinker functions, `P_MobjThinker`, `P_RemoveMobj`, `P_InitThinkers`, `P_AddThinker`, `P_SetThingPosition`, `P_AddActiveCeiling`, `P_AddActivePlat` |
| `doomstat.h` | `players`, `playeringame`, `thinkercap` |
| `r_state.h` | `states`, `sectors`, `sides`, `lines`, `numsectors`, `numlines`, `segs` |
