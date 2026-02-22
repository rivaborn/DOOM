# File Overview

`p_local.h` is the master internal header for the entire play simulation subsystem (`p_` prefix files). It acts as a single inclusion point that exposes all constants, extern variable declarations, and function prototypes used across `p_map.c`, `p_mobj.c`, `p_inter.c`, `p_enemy.c`, `p_user.c`, `p_pspr.c`, `p_tick.c`, `p_maputl.c`, and related files.

Rather than having each `.c` file include a long chain of individual headers, every play file includes `p_local.h` and gets access to the entire local subsystem API. The header also includes `p_spec.h` at the bottom to pull in special sector/line definitions.

The constants defined here capture fundamental movement physics for the engine: block map cell size, player and maximum object radii, gravity, maximum velocity, melee and use distances, and the AI chase persistence threshold.

---

## Global Variables (externs declared)

### Thinker System (from `p_tick.c`)

| Type | Name | Description |
|------|------|-------------|
| `thinker_t` | `thinkercap` | Sentinel node for the doubly-linked thinker list; both the head and tail point to this structure |

### Item Respawn Queue (from `p_mobj.c`)

| Type | Name | Description |
|------|------|-------------|
| `mapthing_t[ITEMQUESIZE]` | `itemrespawnque` | Circular queue of map things waiting to respawn (deathmatch mode) |
| `int[ITEMQUESIZE]` | `itemrespawntime` | Level time at which each queued item was removed |
| `int` | `iquehead` | Write index into the respawn queue |
| `int` | `iquetail` | Read index into the respawn queue |

### Line Opening State (from `p_maputl.c`)

| Type | Name | Description |
|------|------|-------------|
| `fixed_t` | `opentop` | Top of the passable window through a two-sided line |
| `fixed_t` | `openbottom` | Bottom of the passable window through a two-sided line |
| `fixed_t` | `openrange` | Height of the passable window (`opentop - openbottom`) |
| `fixed_t` | `lowfloor` | The lower of the two sector floor heights at a two-sided line; used for drop-off detection |

### Intercept Traversal (from `p_maputl.c`)

| Type | Name | Description |
|------|------|-------------|
| `intercept_t[MAXINTERCEPTS]` | `intercepts` | Fixed-size array of intercept records accumulated during a path traversal |
| `intercept_t*` | `intercept_p` | Next free slot in `intercepts[]` |
| `divline_t` | `trace` | The current path being traced (used by `PIT_Add*Intercepts`) |

### Movement Checking State (from `p_map.c`)

| Type | Name | Description |
|------|------|-------------|
| `boolean` | `floatok` | Set to `true` by `P_CheckPosition` if the move would fit vertically even though something blocked horizontally; lets floating monsters decide whether to float up |
| `fixed_t` | `tmfloorz` | Highest floor height contacted during a position check |
| `fixed_t` | `tmceilingz` | Lowest ceiling height contacted during a position check |
| `line_t*` | `ceilingline` | The line that lowered `tmceilingz` (used by the sky-hack to prevent missiles exploding on sky ceilings) |
| `mobj_t*` | `linetarget` | Set by `P_AimLineAttack` to the mobj that was aimed at (or `NULL`) |

### Map Layout (from `p_setup.c`)

| Type | Name | Description |
|------|------|-------------|
| `byte*` | `rejectmatrix` | The REJECT lump: a bit matrix indexed `[sector_a][sector_b]` used for fast line-of-sight rejection |
| `short*` | `blockmaplump` | The raw BLOCKMAP lump data including the header |
| `short*` | `blockmap` | Pointer into `blockmaplump` past the header; indexed by block number to give the offset into `blockmaplump` for that block's line list |
| `int` | `bmapwidth` | Width of the block map in blocks |
| `int` | `bmapheight` | Height of the block map in blocks |
| `fixed_t` | `bmaporgx` | X origin of the block map in world coordinates |
| `fixed_t` | `bmaporgy` | Y origin of the block map in world coordinates |
| `mobj_t**` | `blocklinks` | Array of mobj list head pointers, one per blockmap cell; used by `P_BlockThingsIterator` |

### Ammo Tables (from `p_inter.c`)

| Type | Name | Description |
|------|------|-------------|
| `int[NUMAMMO]` | `maxammo` | Default maximum ammo capacity per type |
| `int[NUMAMMO]` | `clipammo` | Ammo per clip load per type |

---

## Constants / Macros

### Movement Physics

| Macro | Value | Description |
|-------|-------|-------------|
| `FLOATSPEED` | `FRACUNIT*4` | Units per tic a floating monster moves vertically when adjusting to target height |
| `MAXHEALTH` | `100` | Maximum player health (normal pickups clamp to this) |
| `VIEWHEIGHT` | `41*FRACUNIT` | Eye height above the floor, in fixed-point world units |
| `GRAVITY` | `FRACUNIT` | Downward acceleration applied each tic to non-flying objects |
| `MAXMOVE` | `30*FRACUNIT` | Maximum XY velocity per tic; momenta are clamped to this |

### Block Map

| Macro | Value | Description |
|-------|-------|-------------|
| `MAPBLOCKUNITS` | `128` | Width and height of one block map cell in world units |
| `MAPBLOCKSIZE` | `128*FRACUNIT` | Same as above, in fixed-point |
| `MAPBLOCKSHIFT` | `FRACBITS+7` | Right-shift to convert a fixed-point coordinate to a block index |
| `MAPBMASK` | `MAPBLOCKSIZE-1` | Mask for the fractional part within a block |
| `MAPBTOFRAC` | `MAPBLOCKSHIFT-FRACBITS` | Shift for converting block coordinates back to fractional intercept positions |

### Radii

| Macro | Value | Description |
|-------|-------|-------------|
| `PLAYERRADIUS` | `16*FRACUNIT` | Collision radius of the player |
| `MAXRADIUS` | `32*FRACUNIT` | Maximum collision radius of any moving object (used to expand blockmap queries so objects near cell edges are not missed) |

### Combat Ranges

| Macro | Value | Description |
|-------|-------|-------------|
| `USERANGE` | `64*FRACUNIT` | Maximum distance at which the player can activate a linedef special |
| `MELEERANGE` | `64*FRACUNIT` | Maximum distance for melee attacks |
| `MISSILERANGE` | `32*64*FRACUNIT` | Maximum distance for hitscan attacks (pistol, chaingun, shotgun) |

### AI

| Macro | Value | Description |
|-------|-------|-------------|
| `BASETHRESHOLD` | `100` | Number of tics a monster will pursue its current target exclusively after being damaged by it |

### Mobj Z Placement

| Macro | Value | Description |
|-------|-------|-------------|
| `ONFLOORZ` | `MININT` | Sentinel value for `P_SpawnMobj` z parameter: place the object on the floor |
| `ONCEILINGZ` | `MAXINT` | Sentinel value: place the object hanging from the ceiling |

### Item Queue

| Macro | Value | Description |
|-------|-------|-------------|
| `ITEMQUESIZE` | `128` | Size of the circular item respawn queue |

### Path Traversal Flags

| Macro | Value | Description |
|-------|-------|-------------|
| `PT_ADDLINES` | `1` | Flag for `P_PathTraverse`: add line intercepts |
| `PT_ADDTHINGS` | `2` | Flag for `P_PathTraverse`: add thing intercepts |
| `PT_EARLYOUT` | `4` | Flag for `P_PathTraverse`: stop at the first solid one-sided line |

---

## Functions (prototypes declared)

### Thinker Management (`p_tick.c`)

```c
void P_InitThinkers(void);
void P_AddThinker(thinker_t* thinker);
void P_RemoveThinker(thinker_t* thinker);
```

### Player Sprite System (`p_pspr.c`)

```c
void P_SetupPsprites(player_t* curplayer);
void P_MovePsprites(player_t* curplayer);
void P_DropWeapon(player_t* player);
```

### Player Thinking (`p_user.c`)

```c
void P_PlayerThink(player_t* player);
```

### Map Object Management (`p_mobj.c`)

```c
void P_RespawnSpecials(void);
mobj_t* P_SpawnMobj(fixed_t x, fixed_t y, fixed_t z, mobjtype_t type);
void P_RemoveMobj(mobj_t* th);
boolean P_SetMobjState(mobj_t* mobj, statenum_t state);
void P_MobjThinker(mobj_t* mobj);
void P_SpawnPuff(fixed_t x, fixed_t y, fixed_t z);
void P_SpawnBlood(fixed_t x, fixed_t y, fixed_t z, int damage);
mobj_t* P_SpawnMissile(mobj_t* source, mobj_t* dest, mobjtype_t type);
void P_SpawnPlayerMissile(mobj_t* source, mobjtype_t type);
```

### Monster AI (`p_enemy.c`)

```c
void P_NoiseAlert(mobj_t* target, mobj_t* emmiter);
```

### Map Utilities (`p_maputl.c`)

```c
fixed_t P_AproxDistance(fixed_t dx, fixed_t dy);
int P_PointOnLineSide(fixed_t x, fixed_t y, line_t* line);
int P_PointOnDivlineSide(fixed_t x, fixed_t y, divline_t* line);
void P_MakeDivline(line_t* li, divline_t* dl);
fixed_t P_InterceptVector(divline_t* v2, divline_t* v1);
int P_BoxOnLineSide(fixed_t* tmbox, line_t* ld);
void P_LineOpening(line_t* linedef);
boolean P_BlockLinesIterator(int x, int y, boolean(*func)(line_t*));
boolean P_BlockThingsIterator(int x, int y, boolean(*func)(mobj_t*));
boolean P_PathTraverse(fixed_t x1, fixed_t y1, fixed_t x2, fixed_t y2,
                       int flags, boolean (*trav)(intercept_t*));
void P_UnsetThingPosition(mobj_t* thing);
void P_SetThingPosition(mobj_t* thing);
```

### Movement and Collision (`p_map.c`)

```c
boolean P_CheckPosition(mobj_t* thing, fixed_t x, fixed_t y);
boolean P_TryMove(mobj_t* thing, fixed_t x, fixed_t y);
boolean P_TeleportMove(mobj_t* thing, fixed_t x, fixed_t y);
void P_SlideMove(mobj_t* mo);
boolean P_CheckSight(mobj_t* t1, mobj_t* t2);
void P_UseLines(player_t* player);
boolean P_ChangeSector(sector_t* sector, boolean crunch);
fixed_t P_AimLineAttack(mobj_t* t1, angle_t angle, fixed_t distance);
void P_LineAttack(mobj_t* t1, angle_t angle, fixed_t distance,
                  fixed_t slope, int damage);
void P_RadiusAttack(mobj_t* spot, mobj_t* source, int damage);
```

### Interaction (`p_inter.c`)

```c
void P_TouchSpecialThing(mobj_t* special, mobj_t* toucher);
void P_DamageMobj(mobj_t* target, mobj_t* inflictor,
                  mobj_t* source, int damage);
```

---

## Data Structures

### `divline_t`

Defined in this header. Represents a directed line segment as an origin point plus a delta vector, used in the precise line-side and intercept calculations of the BSP and path-traversal code.

```c
typedef struct {
    fixed_t x;   // origin X
    fixed_t y;   // origin Y
    fixed_t dx;  // direction X component
    fixed_t dy;  // direction Y component
} divline_t;
```

### `intercept_t`

Defined in this header. One record in the `intercepts[]` array built by `P_PathTraverse`. Describes a single intersection point along a trace ray.

```c
typedef struct {
    fixed_t frac;    // fractional distance along the trace (0 = start, FRACUNIT = end)
    boolean isaline; // true if this intercept is with a line; false if with a thing
    union {
        mobj_t* thing;
        line_t* line;
    } d;             // the intersected object
} intercept_t;
```

### `traverser_t`

```c
typedef boolean (*traverser_t)(intercept_t* in);
```

Function pointer type for the callback passed to `P_PathTraverse`. Returns `false` to stop traversal early.

---

## Dependencies

| File | Why needed |
|------|-----------|
| `r_local.h` | Renderer-local types including `sector_t`, `line_t`, `subsector_t`, `side_t`; included conditionally if `__R_LOCAL__` is not already defined |
| `p_spec.h` | Special sector and linedef type definitions included at the bottom of `p_local.h` to complete the local API |
