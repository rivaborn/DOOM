# File Overview

`p_spec.c` is the "special effects" module — one of the largest and most complex gameplay files in DOOM. It handles:

1. **Animated textures and flats** — Cycling through a sequence of WAD lumps to create moving water, lava, and fire effects.
2. **Sector utility functions** — Finding adjacent sector heights and light levels, used throughout floor/ceiling movement code.
3. **Line trigger dispatch** — `P_CrossSpecialLine` handles walk-over triggers; `P_ShootSpecialLine` handles projectile-impact triggers; `P_PlayerInSpecialSector` handles damage and effects for sectors with special types.
4. **Per-frame update** — `P_UpdateSpecials` advances animations, scrolls walls, and counts down button timers.
5. **Level spawn** — `P_SpawnSpecials` creates thinkers for special sectors and sets up scrolling walls.
6. **Donut special** — `EV_DoDonut` implements the complex "donut" floor motion pair.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `anim_t[MAXANIMS]` | `anims` | Array of active animation sequences |
| `anim_t*` | `lastanim` | Pointer one past the last active animation entry |
| `boolean` | `levelTimer` | When `true`, a countdown timer ends the level |
| `int` | `levelTimeCount` | Tics remaining until level timer fires (when `levelTimer == true`) |
| `short` | `numlinespecials` | Number of lines registered for per-tick update (currently only scroll lines) |
| `line_t*[MAXLINEANIMS]` | `linespeciallist` | Array of pointers to lines with per-tick line specials |

## Data Structures

### `anim_t` (local struct)
```c
typedef struct {
    boolean istexture;  // true = wall texture, false = flat
    int     picnum;     // last frame number (current animated index)
    int     basepic;    // first frame number (starting lump)
    int     numpics;    // number of frames in the cycle
    int     speed;      // tics per frame
} anim_t;
```
Describes one running animation sequence. During each tic, `P_UpdateSpecials` computes the current frame index from `leveltime / speed` modulo `numpics`.

### `animdef_t` (local struct)
```c
typedef struct {
    boolean istexture;
    char    endname[9];
    char    startname[9];
    int     speed;
} animdef_t;
```
Source data describing an animation. The global `animdefs[]` array contains the hard-coded animation sequences for all DOOM versions.

### `animdefs[]` (local table)
Hard-coded table of all texture and flat animations for DOOM and DOOM II. Entries include:
- Flats: `NUKAGE1`-`3`, `FWATER1`-`4`, `LAVA1`-`4`, `BLOOD1`-`3`, plus DOOM II floor animations.
- Wall textures: `BLODGR1`-`4`, `SLADRIP1`-`3`, `FIREBLU1`-`2`, `ROCKRED1`-`3`, etc.
Each entry is terminated by a sentinel with `istexture == -1`.

## Functions

### `P_InitPicAnims`
```c
void P_InitPicAnims(void)
```
**Purpose:** Initializes the active animation list by scanning `animdefs[]` and looking up the start/end lump numbers for each defined animation.

**Parameters:** None.

**Key logic:** For each `animdef_t`, checks whether the start lump exists (skips if absent, to handle episode-exclusive textures). Computes `picnum` (last frame) and `basepic` (first frame) via `R_FlatNumForName` or `R_TextureNumForName`. Validates that each animation has at least 2 frames.

### `getSide`
```c
side_t* getSide(int currentSector, int line, int side)
```
**Purpose:** Returns a pointer to a sidedef given a sector index, the sector's line index (within `sector->lines`), and side (0 or 1).

### `getSector`
```c
sector_t* getSector(int currentSector, int line, int side)
```
**Purpose:** Returns the sector on the specified side of a given line of the specified sector.

### `twoSided`
```c
int twoSided(int sector, int line)
```
**Purpose:** Returns non-zero if the specified line of the sector has the `ML_TWOSIDED` flag set.

### `getNextSector`
```c
sector_t* getNextSector(line_t* line, sector_t* sec)
```
**Purpose:** Returns the sector on the opposite side of a two-sided line from `sec`. Returns `NULL` for one-sided lines.

### `P_FindLowestFloorSurrounding`
```c
fixed_t P_FindLowestFloorSurrounding(sector_t* sec)
```
**Purpose:** Returns the lowest floor height among all sectors adjacent to `sec` via two-sided lines. Returns `sec->floorheight` if no neighbors exist.

### `P_FindHighestFloorSurrounding`
```c
fixed_t P_FindHighestFloorSurrounding(sector_t* sec)
```
**Purpose:** Returns the highest floor height among adjacent sectors. Returns `-500*FRACUNIT` if no neighbors.

### `P_FindNextHighestFloor`
```c
fixed_t P_FindNextHighestFloor(sector_t* sec, int currentheight)
```
**Purpose:** Returns the lowest floor height among adjacent sectors that is strictly higher than `currentheight`. Used for stair-building and raise-to-nearest logic.

**Key logic:** Collects all adjacent floor heights greater than `currentheight` into a stack-allocated array (max 20 sectors), then finds the minimum of that set.

### `P_FindLowestCeilingSurrounding`
```c
fixed_t P_FindLowestCeilingSurrounding(sector_t* sec)
```
**Purpose:** Returns the lowest ceiling height among adjacent sectors. Used for raise-floor-to-ceiling logic.

### `P_FindHighestCeilingSurrounding`
```c
fixed_t P_FindHighestCeilingSurrounding(sector_t* sec)
```
**Purpose:** Returns the highest ceiling height among adjacent sectors.

### `P_FindSectorFromLineTag`
```c
int P_FindSectorFromLineTag(line_t* line, int start)
```
**Purpose:** Iterates sectors to find the next sector (after index `start`) whose tag matches `line->tag`. Returns -1 when exhausted. Used by all tag-based trigger functions.

### `P_FindMinSurroundingLight`
```c
int P_FindMinSurroundingLight(sector_t* sector, int max)
```
**Purpose:** Returns the minimum light level among adjacent sectors, clamped to a starting maximum. Used by light strobe effects.

### `P_CrossSpecialLine`
```c
void P_CrossSpecialLine(int linenum, int side, mobj_t* thing)
```
**Purpose:** Called every time a thing's origin crosses a linedef with a non-zero special. Dispatches the appropriate action.

**Parameters:**
- `linenum` - Index of the crossed line.
- `side` - Which side the crossing occurred from.
- `thing` - The crossing object.

**Key logic:**
- Non-player things are filtered: only specific special numbers (teleports, plat, raise door) can be activated by monsters; others are ignored.
- Projectile types (rockets, plasma, BFG shots, etc.) are always excluded.
- Single-trigger specials have their `line->special` zeroed after activation (consumed); re-trigger specials do not.
- The large `switch` handles all crossing-triggered line specials (approximately 80 case values), calling `EV_DoDoor`, `EV_DoFloor`, `EV_DoCeiling`, `EV_DoPlat`, `EV_BuildStairs`, `EV_Teleport`, `G_ExitLevel`, etc.

### `P_ShootSpecialLine`
```c
void P_ShootSpecialLine(mobj_t* thing, line_t* line)
```
**Purpose:** Called when a hitscan attack (bullet or BFG tracer) hits a special line. Dispatches impact-triggered actions.

**Key logic:** Only a small number of impact specials are defined: case 24 (raise floor), case 46 (open door, also monster-activatable), case 47 (raise platform to nearest). Each calls `P_ChangeSwitchTexture` after the action.

### `P_PlayerInSpecialSector`
```c
void P_PlayerInSpecialSector(player_t* player)
```
**Purpose:** Called each tic when a player is standing on a sector with a non-zero special. Applies environmental effects.

**Key logic:**
- Returns early if the player's z is not at floor height (still falling).
- Sector special 5: hellslime — 10 damage every 32 tics unless wearing iron feet.
- Sector special 7: nukage — 5 damage every 32 tics.
- Sector specials 4 and 16: strobe hurt / super hellslime — 20 damage even with iron feet (5% chance bypasses protection for type 4).
- Sector special 9: secret sector — increments `player->secretcount` and clears the special.
- Sector special 11: exit damage — removes god mode, 20 damage every 32 tics, exits level when health <= 10.

### `P_UpdateSpecials`
```c
void P_UpdateSpecials(void)
```
**Purpose:** Called once per game tic to advance all animations, scroll textures, and count down buttons.

**Key logic:**
1. **Level timer**: Decrements `levelTimeCount`; calls `G_ExitLevel` when it reaches zero.
2. **Flat/texture animation**: For each active animation, computes the current frame via `(leveltime / speed + i) % numpics` and updates `texturetranslation[]` or `flattranslation[]` for every lump index in the range. This redirection makes all references to any frame of the animation use the current frame.
3. **Line scroll**: For each line in `linespeciallist[]` with special 48, increments `sides[sidenum[0]].textureoffset` by `FRACUNIT` (one pixel/tic horizontal scroll).
4. **Button timers**: For each entry in `buttonlist[]` with nonzero `btimer`, decrements it. When it reaches zero, restores the original switch texture to the appropriate wall tier (top/middle/bottom) and plays the switch sound.

### `EV_DoDonut`
```c
int EV_DoDonut(line_t* line)
```
**Purpose:** Implements the "donut" floor action. For each sector matching the line's tag: the inner ring (s1) lowers to match the outer ring (s3), while the middle ring (s2) rises to match s3's floor height and adopts s3's floor texture.

**Return value:** 1 if any donut action was started; 0 otherwise.

### `P_SpawnSpecials`
```c
void P_SpawnSpecials(void)
```
**Purpose:** Called after map load. Scans all sectors for special types and spawns the corresponding thinkers. Also processes `-timer` and `-avg` command-line options.

**Key logic:**
- `-avg` sets a 20-minute level timer (20 * 60 * 35 tics).
- `-timer` sets a custom level timer from the command line argument.
- Sector specials spawn: light flash (1), fast strobe (2), slow strobe (3), fast strobe+damage (4), glow (8), door-close-in-30 (10), sync strobe slow (12), sync strobe fast (13), door-raise-in-5-minutes (14), fire flicker (17).
- Sector special 9 (secret) simply increments `totalsecret` — no thinker needed.
- Scans lines for special 48 (scroll) and populates `linespeciallist[]`.
- Zeros `activeceilings[]`, `activeplats[]`, and `buttonlist[]`.

## Dependencies

| File | Reason |
|------|--------|
| `doomdef.h` | Constants and type definitions |
| `doomstat.h` | `leveltime`, `totalsecret`, `deathmatch` |
| `i_system.h` | `I_Error` |
| `z_zone.h` | `Z_Malloc` for donut thinkers |
| `m_argv.h` | `M_CheckParm` for `-timer`/`-avg` |
| `m_random.h` | `P_Random` for damage chance in sector special 4 |
| `w_wad.h` | `W_CheckNumForName` for episode detection |
| `r_local.h` | `texturetranslation`, `flattranslation` |
| `p_local.h` | `P_DamageMobj`, `P_SpawnLightFlash`, thinker/mover functions |
| `g_game.h` | `G_ExitLevel`, `G_SecretExitLevel` |
| `s_sound.h` | `S_StartSound` for button sounds |
| `r_state.h` | `texturetranslation`, `flattranslation` |
| `sounds.h` | `sfx_swtchn` sound constant |
