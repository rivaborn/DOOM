# File Overview

`p_spec.h` is the master header for DOOM's special effects subsystem. It brings together declarations for five sub-modules that were originally separate but are all included via this single header:

- `p_spec.c` — General special effects, animation, trigger dispatch
- `p_lights.c` — Light effect thinkers (flicker, strobe, glow, flash)
- `p_switch.c` — Switch and button texture management
- `p_plats.c` — Platform/lift thinkers
- `p_doors.c` — Door thinkers
- `p_ceilng.c` — Ceiling thinkers
- `p_floor.c` — Floor movement thinkers
- `p_telept.c` — Teleportation

The header defines all the data structures for these subsystems and declares all their public functions in one place.

## Global Variables (extern)

| Type | Name | Description |
|------|------|-------------|
| `boolean` | `levelTimer` | Whether a countdown timer is active (from `-timer`/`-avg`) |
| `int` | `levelTimeCount` | Remaining tics for the level timer |
| `button_t[MAXBUTTONS]` | `buttonlist` | Active button press records; defined in `p_switch.c` |
| `plat_t*[MAXPLATS]` | `activeplats` | Registered active platforms; defined in `p_plats.c` |
| `ceiling_t*[MAXCEILINGS]` | `activeceilings` | Registered active ceilings; defined in `p_ceilng.c` |

## Data Structures

### `fireflicker_t`
```c
typedef struct {
    thinker_t thinker;
    sector_t* sector;
    int count;
    int maxlight;
    int minlight;
} fireflicker_t;
```
Thinker for fire-flickering light effect (sector special 17). Randomly alternates between max and min light levels.

### `lightflash_t`
```c
typedef struct {
    thinker_t thinker;
    sector_t* sector;
    int count;
    int maxlight;
    int minlight;
    int maxtime;
    int mintime;
} lightflash_t;
```
Thinker for random light flash effect (sector special 1). Uses random timing intervals.

### `strobe_t`
```c
typedef struct {
    thinker_t thinker;
    sector_t* sector;
    int count;
    int minlight;
    int maxlight;
    int darktime;
    int brighttime;
} strobe_t;
```
Thinker for strobe light effect (sector specials 2, 3, 4, 12, 13). Alternates between `minlight` and `maxlight` at fixed intervals.

### `glow_t`
```c
typedef struct {
    thinker_t thinker;
    sector_t* sector;
    int minlight;
    int maxlight;
    int direction;
} glow_t;
```
Thinker for smoothly oscillating light effect (sector special 8). `direction` is +1 (brightening) or -1 (darkening).

### `switchlist_t`
```c
typedef struct {
    char  name1[9];
    char  name2[9];
    short episode;
} switchlist_t;
```
Describes a switch texture pair (inactive name / active name) and the episode it belongs to.

### `bwhere_e`
```c
typedef enum { top, middle, bottom } bwhere_e;
```
Which texture tier of a wall a button press applies to.

### `button_t`
```c
typedef struct {
    line_t*   line;
    bwhere_e  where;
    int       btexture;
    int       btimer;
    mobj_t*   soundorg;
} button_t;
```
Active button record. `btimer` counts down from `BUTTONTIME` (35 tics = 1 second); when it reaches zero, the original texture is restored and a click sound plays.

### `plat_e`
```c
typedef enum { up, down, waiting, in_stasis } plat_e;
```
State of a platform thinker.

### `plattype_e`
```c
typedef enum { perpetualRaise, downWaitUpStay, raiseAndChange, raiseToNearestAndChange, blazeDWUS } plattype_e;
```
Platform behavior type.

### `plat_t`
```c
typedef struct {
    thinker_t thinker;
    sector_t* sector;
    fixed_t   speed;
    fixed_t   low;
    fixed_t   high;
    int       wait;
    int       count;
    plat_e    status;
    plat_e    oldstatus;
    boolean   crush;
    int       tag;
    plattype_e type;
} plat_t;
```
Platform (lift) thinker. `low`/`high` define the floor height range; `count` is the wait timer.

### `vldoor_e`
```c
typedef enum {
    normal, close30ThenOpen, close, open,
    raiseIn5Mins, blazeRaise, blazeOpen, blazeClose
} vldoor_e;
```
Door type. `blazeRaise/Open/Close` are faster variants.

### `vldoor_t`
```c
typedef struct {
    thinker_t thinker;
    vldoor_e  type;
    sector_t* sector;
    fixed_t   topheight;
    fixed_t   speed;
    int       direction;    // 1=up, 0=waiting, -1=down
    int       topwait;
    int       topcountdown;
} vldoor_t;
```
Door thinker. `topheight` is the fully-open ceiling height.

### `ceiling_e`
```c
typedef enum {
    lowerToFloor, raiseToHighest, lowerAndCrush,
    crushAndRaise, fastCrushAndRaise, silentCrushAndRaise
} ceiling_e;
```
Ceiling movement type.

### `ceiling_t`
```c
typedef struct {
    thinker_t thinker;
    ceiling_e type;
    sector_t* sector;
    fixed_t   bottomheight;
    fixed_t   topheight;
    fixed_t   speed;
    boolean   crush;
    int       direction;   // 1=up, 0=waiting, -1=down
    int       tag;
    int       olddirection;
} ceiling_t;
```
Ceiling thinker. `crush` causes damage to things when the ceiling descends into them.

### `floor_e`
```c
typedef enum {
    lowerFloor, lowerFloorToLowest, turboLower, raiseFloor,
    raiseFloorToNearest, raiseToTexture, lowerAndChange,
    raiseFloor24, raiseFloor24AndChange, raiseFloorCrush,
    raiseFloorTurbo, donutRaise, raiseFloor512
} floor_e;
```
Floor movement type.

### `stair_e`
```c
typedef enum { build8, turbo16 } stair_e;
```
Stair-building speed.

### `floormove_t`
```c
typedef struct {
    thinker_t thinker;
    floor_e   type;
    boolean   crush;
    sector_t* sector;
    int       direction;
    int       newspecial;
    short     texture;
    fixed_t   floordestheight;
    fixed_t   speed;
} floormove_t;
```
Floor movement thinker.

### `result_e`
```c
typedef enum { ok, crushed, pastdest } result_e;
```
Return type from `T_MovePlane`: `ok` (moving), `crushed` (hit something), `pastdest` (reached destination).

## Constants

| Name | Value | Description |
|------|-------|-------------|
| `MO_TELEPORTMAN` | 14 | Map object type number for the teleport destination spot |
| `MAXSWITCHES` | 50 | Maximum number of switch texture pairs |
| `MAXBUTTONS` | 16 | Maximum simultaneous button presses (4 players * 4 buttons) |
| `BUTTONTIME` | 35 | Button reversion time in tics (1 second) |
| `GLOWSPEED` | 8 | Light level change per tic for glow effect |
| `STROBEBRIGHT` | 5 | Bright duration in tics for strobe |
| `FASTDARK` | 15 | Dark duration for fast strobe |
| `SLOWDARK` | 35 | Dark duration for slow strobe |
| `PLATWAIT` | 3 | Platform wait time in seconds |
| `PLATSPEED` | `FRACUNIT` | Default platform speed |
| `MAXPLATS` | 30 | Maximum active platforms |
| `VDOORSPEED` | `FRACUNIT*2` | Default door speed |
| `VDOORWAIT` | 150 | Door wait time in tics |
| `CEILSPEED` | `FRACUNIT` | Default ceiling movement speed |
| `CEILWAIT` | 150 | Ceiling wait tics |
| `MAXCEILINGS` | 30 | Maximum active ceilings |
| `FLOORSPEED` | `FRACUNIT` | Default floor movement speed |

## Dependencies

| File | Reason |
|------|--------|
| (consumers must include) | `doomdef.h` for `boolean`, `fixed_t`, `mobj_t` definitions |
