# File Overview

`p_setup.c` is the map loader and level setup module. It is responsible for reading all map data lumps from the WAD file, converting them from their on-disk format into the runtime data structures used by the game, and performing the post-load computations that bind the geometry together. It also handles initializing global game-subsystem state before a level begins.

DOOM maps are stored in the WAD file as a set of numbered lumps (VERTEXES, SEGS, SECTORS, SSECTORS, NODES, LINEDEFS, SIDEDEFS, BLOCKMAP, REJECT, THINGS) following a map-name marker lump (e.g., `E1M1` or `MAP01`). `p_setup.c` defines a loader for each lump type and calls them all from `P_SetupLevel`.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `int` | `numvertexes` | Count of loaded vertices |
| `vertex_t*` | `vertexes` | Array of all map vertices |
| `int` | `numsegs` | Count of loaded segs |
| `seg_t*` | `segs` | Array of all BSP line segments |
| `int` | `numsectors` | Count of loaded sectors |
| `sector_t*` | `sectors` | Array of all map sectors |
| `int` | `numsubsectors` | Count of loaded subsectors (BSP leaves) |
| `subsector_t*` | `subsectors` | Array of all BSP leaf nodes |
| `int` | `numnodes` | Count of loaded BSP nodes |
| `node_t*` | `nodes` | Array of all BSP internal nodes |
| `int` | `numlines` | Count of loaded linedefs |
| `line_t*` | `lines` | Array of all map linedefs |
| `int` | `numsides` | Count of loaded sidedefs |
| `side_t*` | `sides` | Array of all map sidedefs |
| `int` | `bmapwidth` | Width of the blockmap in 128-unit blocks |
| `int` | `bmapheight` | Height of the blockmap in 128-unit blocks |
| `short*` | `blockmap` | Pointer into `blockmaplump` at offset 4 (past header) |
| `short*` | `blockmaplump` | Raw blockmap lump data including 4-short header (orgx, orgy, w, h) |
| `fixed_t` | `bmaporgx` | X origin of the blockmap in fixed-point map units |
| `fixed_t` | `bmaporgy` | Y origin of the blockmap in fixed-point map units |
| `mobj_t**` | `blocklinks` | Array of mobj chain head pointers, one per blockmap cell |
| `byte*` | `rejectmatrix` | REJECT table lump: a bit array of sector-pair visibility flags |
| `mapthing_t[MAX_DEATHMATCH_STARTS]` | `deathmatchstarts` | Pool of deathmatch spawn points |
| `mapthing_t*` | `deathmatch_p` | Next available deathmatch spawn slot |
| `mapthing_t[MAXPLAYERS]` | `playerstarts` | Player cooperative start positions |

## Functions

### `P_LoadVertexes`
```c
void P_LoadVertexes(int lump)
```
**Purpose:** Loads the VERTEXES lump and converts map coordinates (short integers in map units) to fixed-point `vertex_t` values.

**Parameters:**
- `lump` - WAD lump number of the VERTEXES data.

**Key logic:** Reads each `mapvertex_t` (two 16-bit signed shorts), applies `SHORT()` byte-swapping, and shifts left by `FRACBITS` (16) to convert to fixed-point.

### `P_LoadSegs`
```c
void P_LoadSegs(int lump)
```
**Purpose:** Loads the SEGS lump and resolves all internal cross-references.

**Parameters:**
- `lump` - WAD lump number.

**Key logic:** For each `mapseg_t`, resolves `v1`/`v2` indices into `vertex_t*`, converts angle and offset to fixed-point, resolves `linedef` to a `line_t*`, resolves side number to a `sidedef*`, and sets `frontsector`/`backsector` by following the linedef's sidenum array. Two-sided segs get their backsector from `sidenum[side^1]`.

### `P_LoadSubsectors`
```c
void P_LoadSubsectors(int lump)
```
**Purpose:** Loads the SSECTORS (subsectors) lump, which describes BSP leaf nodes.

**Parameters:**
- `lump` - WAD lump number.

**Key logic:** Each `mapsubsector_t` contains `numsegs` and `firstseg` (index into the segs array). These are stored directly in `subsector_t`. The sector reference is filled in later by `P_GroupLines`.

### `P_LoadSectors`
```c
void P_LoadSectors(int lump)
```
**Purpose:** Loads the SECTORS lump.

**Parameters:**
- `lump` - WAD lump number.

**Key logic:** Converts floor/ceiling heights from short map units to fixed-point. Resolves floor/ceiling texture names via `R_FlatNumForName`. Stores light level, special type, and tag. Initializes `thinglist` to NULL.

### `P_LoadNodes`
```c
void P_LoadNodes(int lump)
```
**Purpose:** Loads the NODES lump (BSP tree internal nodes).

**Parameters:**
- `lump` - WAD lump number.

**Key logic:** Each `mapnode_t` becomes a `node_t`. The partition line (x, y, dx, dy) and child bounding boxes are converted to fixed-point. Child indices are stored as-is (high bit set = subsector index).

### `P_LoadThings`
```c
void P_LoadThings(int lump)
```
**Purpose:** Loads the THINGS lump and spawns map objects.

**Parameters:**
- `lump` - WAD lump number.

**Key logic:** Filters out DOOM II-exclusive monster types (Archvile, Arachnotron, Mancubus, etc.) when not in commercial mode. For each remaining thing, calls `P_SpawnMapThing` to instantiate the game object.

### `P_LoadLineDefs`
```c
void P_LoadLineDefs(int lump)
```
**Purpose:** Loads the LINEDEFS lump.

**Parameters:**
- `lump` - WAD lump number.

**Key logic:** For each `maplinedef_t`: resolves vertex indices, computes `dx`/`dy` delta, classifies slope type (`ST_HORIZONTAL`, `ST_VERTICAL`, `ST_POSITIVE`, `ST_NEGATIVE`), computes the bounding box, resolves side indices, and sets `frontsector`/`backsector` by following the side indices.

### `P_LoadSideDefs`
```c
void P_LoadSideDefs(int lump)
```
**Purpose:** Loads the SIDEDEFS lump.

**Parameters:**
- `lump` - WAD lump number.

**Key logic:** For each `mapsidedef_t`: converts texture offsets to fixed-point, resolves texture name strings via `R_TextureNumForName`, and resolves the sector index.

### `P_LoadBlockMap`
```c
void P_LoadBlockMap(int lump)
```
**Purpose:** Loads the BLOCKMAP lump and initializes the mobj chain array.

**Parameters:**
- `lump` - WAD lump number.

**Key logic:** Caches the lump at `PU_LEVEL`. Applies `SHORT()` byte-swap to every entry. Extracts the four-short header (origin x/y, width, height) into the `bmap*` globals. Allocates and zero-initializes `blocklinks` (one `mobj_t*` per cell).

### `P_GroupLines`
```c
void P_GroupLines(void)
```
**Purpose:** Builds sector-to-line lookup tables and sector bounding boxes.

**Parameters:** None.

**Key logic:**
1. Walks all subsectors and sets `subsector->sector` by following `segs[firstline].sidedef->sector`.
2. Counts how many times each sector appears as a front or back sector of a linedef.
3. Allocates a global line pointer buffer and fills `sector->lines[]` with all lines touching each sector.
4. Computes per-sector `soundorg` at the center of the bounding box (for positional audio).
5. Converts the bounding box to blockmap block coordinates, clamped to blockmap bounds, stored in `sector->blockbox`.

### `P_SetupLevel`
```c
void P_SetupLevel(int episode, int map, int playermask, skill_t skill)
```
**Purpose:** Master level initialization function. Frees old level data, loads the new level, and sets up all subsystems.

**Parameters:**
- `episode` - Episode number (1–4 for registered DOOM, unused for DOOM II).
- `map` - Map number within the episode.
- `playermask` - Bitmask of players in game (used for network, not directly here).
- `skill` - Skill level (affects monster behavior via `P_SpawnMapThing`).

**Key logic:**
1. Resets stats (totalkills, totalitems, totalsecret) and player counts.
2. Stops sounds via `S_Start()`.
3. Frees zone memory down to `PU_LEVEL` tag.
4. Calls `P_InitThinkers()`.
5. Constructs the lump name string (`E1M1` style or `MAP01` style).
6. Calls all `P_Load*` functions in dependency order (blockmap first, then vertexes, sectors, sides, lines, subsectors, nodes, segs — order matters because each loader references previously-loaded arrays).
7. Caches the REJECT lump.
8. Calls `P_GroupLines()`.
9. Loads things and spawns deathmatch players if needed.
10. Calls `P_SpawnSpecials()` to activate sector specials.
11. Optionally calls `R_PrecacheLevel()`.

### `P_Init`
```c
void P_Init(void)
```
**Purpose:** One-time initialization of the play subsystem at game startup.

**Parameters:** None.

**Key logic:** Calls `P_InitSwitchList()`, `P_InitPicAnims()`, and `R_InitSprites(sprnames)` to set up switch textures, animated textures, and sprite definitions.

## Dependencies

| File | Reason |
|------|--------|
| `z_zone.h` | Memory allocation (`Z_Malloc`, `Z_FreeTags`) |
| `m_swap.h` | `SHORT()` / `LONG()` byte-swap macros for cross-platform WAD reading |
| `m_bbox.h` | `M_ClearBox`, `M_AddToBox` for bounding box computation |
| `g_game.h` | `G_DeathMatchSpawnPlayer`, `wminfo` struct |
| `i_system.h` | `I_Error` |
| `w_wad.h` | `W_CacheLumpNum`, `W_LumpLength`, `W_GetNumForName`, `W_Reload` |
| `doomdef.h` | Constants (`FRACBITS`, `MAPBLOCKSHIFT`, etc.) |
| `p_local.h` | `P_InitThinkers`, `P_SpawnMapThing`, `P_SpawnSpecials`, `P_InitSwitchList`, `P_InitPicAnims` |
| `s_sound.h` | `S_Start` to stop sounds before level free |
| `doomstat.h` | Global game state variables |
| `r_data.h` | `R_FlatNumForName`, `R_TextureNumForName`, `R_PrecacheLevel`, `R_InitSprites` |
