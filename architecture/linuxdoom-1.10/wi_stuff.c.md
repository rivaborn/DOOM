# File Overview

`wi_stuff.c` implements the DOOM intermission screen — the screen displayed between levels after the player exits a map and before the next map begins. The intermission screen shows end-of-level statistics (kills, items, secrets, and time), plays appropriate music, animates graphical elements on a world map background (for episode-based DOOM), and provides a state-machine to sequence through the different display phases.

The intermission system supports three game modes:
- **Single-player**: Shows kill/item/secret percentages and level time, with statistics revealed progressively (counting up) for dramatic effect.
- **Cooperative netgame**: Shows per-player kill/item/secret percentages and optionally frag counts.
- **Deathmatch**: Shows a kill matrix (who killed whom) with rolling totals.

For the registered/retail DOOM episodes (not DOOM II), the intermission also shows an episode world map with animated background elements and a "You Are Here" pointer on the next level's location.

The statistics are revealed in a "ticker" animation: values count up from 0 to their final values over several seconds, with pistol sound effects playing on each increment and an explosion sound when each category finalizes. Players can press fire or use to skip the animation.

---

## Global Variables (static — internal to this module)

### State and Timing

| Variable | Type | Purpose |
|----------|------|---------|
| `acceleratestage` | `static int` | Set to 1 when the player presses fire/use to skip the current counting animation. Cleared at each state transition. |
| `me` | `static int` | The console player number (`wbs->pnum`). Used to highlight the local player's row in multiplayer stats. |
| `state` | `static stateenum_t` | Current intermission state machine state: `StatCount` (showing statistics), `ShowNextLoc` (showing world map with next level pointer), or `NoState` (brief pause before exiting). |
| `wbs` | `static wbstartstruct_t*` | Pointer to the intermission data structure passed in by `G_WorldDone`. Contains episode/map numbers, player stats, par time, and max counts. |
| `plrs` | `static wbplayerstruct_t*` | Shortcut pointer to `wbs->plyr[]`, the per-player statistics array. |
| `cnt` | `static int` | General-purpose countdown timer, used to control transition timing between states. |
| `bcnt` | `static int` | Background animation frame counter. Incremented every tic; used to drive animated background patches. |
| `firstrefresh` | `static int` | Flag set on initialization; signals that the first frame needs a full redraw. |

### Per-Player Counters (displayed statistics)

| Variable | Type | Purpose |
|----------|------|---------|
| `cnt_kills[MAXPLAYERS]` | `static int[]` | Currently displayed kill percentages (count up from 0 to actual during animation). |
| `cnt_items[MAXPLAYERS]` | `static int[]` | Currently displayed item percentages. |
| `cnt_secret[MAXPLAYERS]` | `static int[]` | Currently displayed secret percentages. |
| `cnt_time` | `static int` | Currently displayed level time in seconds. |
| `cnt_par` | `static int` | Currently displayed par time in seconds. |
| `cnt_pause` | `static int` | Pause counter used between stat-reveal phases (waits one TICRATE before showing next category). |
| `NUMCMAPS` | `static int` | Number of commercial-mode level names to load (32 for DOOM II). |

### Deathmatch-Specific

| Variable | Type | Purpose |
|----------|------|---------|
| `dm_state` | `static int` | Deathmatch stats reveal sub-state (1-4). |
| `dm_frags[MAXPLAYERS][MAXPLAYERS]` | `static int[][]` | Currently displayed frag counts per killer/victim pair (count toward actual values). |
| `dm_totals[MAXPLAYERS]` | `static int[]` | Currently displayed total frag counts per player. |

### Netgame-Specific

| Variable | Type | Purpose |
|----------|------|---------|
| `cnt_frags[MAXPLAYERS]` | `static int[]` | Displayed frag totals for co-op/netgame mode. |
| `dofrags` | `static int` | Boolean: non-zero if any player has frags, causing the frags column to be displayed. |
| `ng_state` | `static int` | Netgame stats reveal sub-state (1-10). |

### Single-Player-Specific

| Variable | Type | Purpose |
|----------|------|---------|
| `sp_state` | `static int` | Single-player stats reveal sub-state (1-10). |
| `snl_pointeron` | `static boolean` | Controls whether the "You Are Here" pointer flashes on the world map (toggled every 20 tics). |

### Graphics Patches

| Variable | Type | Purpose |
|----------|------|---------|
| `bg` | `static patch_t*` | The background world map image (`WIMAP0`/`WIMAP1`/`WIMAP2` or `INTERPIC`). |
| `yah[2]` | `static patch_t*[]` | "You Are Here" arrow graphics (two alternating versions for flashing). |
| `splat` | `static patch_t*` | Splat graphic marking completed levels on the world map. |
| `percent` | `static patch_t*` | The `%` sign graphic (`WIPCNT`). |
| `colon` | `static patch_t*` | The `:` separator for times (`WICOLON`). |
| `num[10]` | `static patch_t*[]` | Digit graphics 0-9 (`WINUM0` through `WINUM9`). |
| `wiminus` | `static patch_t*` | Minus sign graphic (`WIMINUS`). |
| `finished` | `static patch_t*` | "Finished!" banner graphic (`WIF`). |
| `entering` | `static patch_t*` | "Entering" banner graphic (`WIENTER`). |
| `sp_secret` | `static patch_t*` | Long "Secret" label (`WISCRT2`). |
| `kills` | `static patch_t*` | "Kills" column header (`WIOSTK`). |
| `secret` | `static patch_t*` | Short "Scrt" label for multiplayer (`WIOSTS`). |
| `items` | `static patch_t*` | "Items" column header (`WIOSTI`). |
| `frags` | `static patch_t*` | "Frgs" column header (`WIFRGS`). |
| `time` | `static patch_t*` | "Time" label (`WITIME`). |
| `par` | `static patch_t*` | "Par" label (`WIPAR`). |
| `sucks` | `static patch_t*` | "Sucks" label for times exceeding 1 hour (`WISUCKS`). |
| `killers` | `static patch_t*` | "Killers" label for deathmatch matrix (`WIKILRS`). |
| `victims` | `static patch_t*` | "Victims" label for deathmatch matrix (`WIVCTMS`). |
| `total` | `static patch_t*` | "Total" label for deathmatch matrix (`WIMSTT`). |
| `star` | `static patch_t*` | Player's live face graphic (used to mark the local player). |
| `bstar` | `static patch_t*` | Player's dead face graphic. |
| `p[MAXPLAYERS]` | `static patch_t*[]` | Player number graphics (colored) for multiplayer stat displays. |
| `bp[MAXPLAYERS]` | `static patch_t*[]` | Gray player number graphics (inactive players). |
| `lnames` | `static patch_t**` | Array of level name graphics (WILVxy for episodes, CWILVxx for DOOM II). Heap-allocated. |

### Static Animation Data

| Variable | Type | Purpose |
|----------|------|---------|
| `lnodes[NUMEPISODES][NUMMAPS]` | `static point_t[][]` | Pixel coordinates on the world map background where each level's node (splat/pointer) is placed. Hard-coded for each of the 3 episodes × 9 maps. |
| `epsd0animinfo[]` | `static anim_t[]` | Animation definitions for episode 1's world map (10 always-cycling animations). |
| `epsd1animinfo[]` | `static anim_t[]` | Animation definitions for episode 2's world map (9 level-trigger animations, shown when the next level is reached). |
| `epsd2animinfo[]` | `static anim_t[]` | Animation definitions for episode 3's world map (6 always-cycling animations). |
| `NUMANIMS[NUMEPISODES]` | `static int[]` | Count of animations for each episode (10, 9, 6). |
| `anims[NUMEPISODES]` | `static anim_t*[]` | Pointers to the per-episode animation definition arrays. |

---

## Functions

### Background and Drawing Utilities

#### `WI_slamBackground`

```c
void WI_slamBackground(void);
```

**Purpose:** Copies screen 1 (the pre-rendered background) to screen 0 (the display buffer), effectively resetting the displayed image to the background without any overlaid statistics.

**Key logic:** `memcpy(screens[0], screens[1], SCREENWIDTH * SCREENHEIGHT)`, followed by `V_MarkRect` to mark the whole screen dirty.

---

#### `WI_Responder`

```c
boolean WI_Responder(event_t* ev);
```

**Purpose:** Event handler stub. The intermission screen does not directly process keyboard/mouse events — instead, button presses are detected in `WI_checkForAccelerate` by polling player command buttons each tic.

**Return value:** Always `false` (event not consumed).

---

#### `WI_drawLF`

```c
void WI_drawLF(void);
```

**Purpose:** Draws the "[LevelName] Finished!" banner at the top of the screen.

**Key logic:** Draws `lnames[wbs->last]` centered horizontally at y=`WI_TITLEY`, then draws the `finished` patch below it (spaced by 5/4 of the level name's height).

---

#### `WI_drawEL`

```c
void WI_drawEL(void);
```

**Purpose:** Draws the "Entering [LevelName]" banner.

**Key logic:** Draws the `entering` patch centered, then draws `lnames[wbs->next]` below it.

---

#### `WI_drawOnLnode`

```c
void WI_drawOnLnode(int n, patch_t* c[]);
```

**Purpose:** Draws a patch (from an array of alternatives) at the world map location for level `n`. Tries each alternative in the array until one fits within the screen boundaries.

**Parameters:**
- `n` - Level number (0-8).
- `c` - Array of two patch alternatives to try.

**Key logic:** Computes the bounding box for each patch at `lnodes[wbs->epsd][n]`. Uses the first one that fits on screen. If neither fits, prints a debug message.

---

### Background Animation

#### `WI_initAnimatedBack`

```c
void WI_initAnimatedBack(void);
```

**Purpose:** Initializes the per-animation state for the current episode's background animations. Called when entering `ShowNextLoc` state.

**Key logic:** For each animation in `anims[wbs->epsd][]`, resets `ctr = -1` and sets `nexttic` based on the animation type (`ANIM_ALWAYS`: random start within period; `ANIM_RANDOM`: random base + deviation; `ANIM_LEVEL`: immediate start).

---

#### `WI_updateAnimatedBack`

```c
void WI_updateAnimatedBack(void);
```

**Purpose:** Advances all background animations by one tic. Called every tic during states that show the world map.

**Key logic:** For each animation in the current episode, checks if `bcnt == a->nexttic`. If so, advances the frame counter according to the animation type:
- `ANIM_ALWAYS`: cycles frames continuously.
- `ANIM_RANDOM`: plays a sequence then waits a random interval.
- `ANIM_LEVEL`: plays frames only when `wbs->next` matches `a->data1` (the level this animation is associated with).

---

#### `WI_drawAnimatedBack`

```c
void WI_drawAnimatedBack(void);
```

**Purpose:** Draws the current frame of each active background animation.

**Key logic:** For each animation with `ctr >= 0`, calls `V_DrawPatch(a->loc.x, a->loc.y, FB, a->p[a->ctr])`.

---

### Number and Time Drawing

#### `WI_drawNum`

```c
int WI_drawNum(int x, int y, int n, int digits);
```

**Purpose:** Draws an integer `n` as a right-aligned series of digit patches, ending at x-coordinate `x`.

**Parameters:**
- `x` - Right edge x-coordinate (digits are placed working leftward from here).
- `y` - Y-coordinate.
- `n` - The number to draw.
- `digits` - Minimum number of digits to show. If negative, computes the natural digit count automatically.

**Return value:** The new x coordinate after drawing (left edge of the number including minus sign).

**Key logic:** Special case: if `n == 1994`, does not draw anything (the "1994" sentinel means "not applicable"). Draws digits right-to-left using `num[n%10]` patches, prepending a minus sign if negative.

---

#### `WI_drawPercent`

```c
void WI_drawPercent(int x, int y, int p);
```

**Purpose:** Draws a percentage value: draws the `%` sign at `x`, then `WI_drawNum` to the left of it.

**Parameters:**
- `x` - X position of the `%` sign.
- `y` - Y position.
- `p` - Percentage value. If negative, nothing is drawn.

---

#### `WI_drawTime`

```c
void WI_drawTime(int x, int y, int t);
```

**Purpose:** Draws a time value in MM:SS format. If the time exceeds 61 minutes 59 seconds (3719 seconds), draws the "Sucks" graphic instead (the level is considered too long to display meaningfully).

**Parameters:**
- `x` - Right edge x-coordinate for the time display.
- `y` - Y-coordinate.
- `t` - Time in seconds.

**Key logic:** Uses `WI_drawNum` and colon patches to build MM:SS or HH:MM:SS displays. The sentinel value `t < 0` causes nothing to be drawn (used for uninitialized/not-applicable times).

---

### State Machine

The intermission uses a nested state machine. The outer states are `StatCount`, `ShowNextLoc`, and `NoState`. Within `StatCount`, sub-states (`sp_state`, `ng_state`, `dm_state`) control which statistics category is currently counting up.

#### `WI_End`

```c
void WI_End(void);
```

**Purpose:** Cleans up the intermission by calling `WI_unloadData`.

---

#### `WI_initNoState` / `WI_updateNoState`

```c
void WI_initNoState(void);
void WI_updateNoState(void);
```

**Purpose:** `NoState` is a brief 10-tic pause before calling `G_WorldDone` to start the next level. `WI_updateNoState` decrements `cnt` and calls `G_WorldDone` when it reaches zero.

---

#### `WI_initShowNextLoc` / `WI_updateShowNextLoc` / `WI_drawShowNextLoc`

```c
void WI_initShowNextLoc(void);
void WI_updateShowNextLoc(void);
void WI_drawShowNextLoc(void);
```

**Purpose:** The `ShowNextLoc` state displays the world map with completed levels splatted and a flashing "You Are Here" pointer on the next level. Active for `SHOWNEXTLOCDELAY * TICRATE = 4 * 35 = 140` tics (4 seconds), or until the player presses fire/use.

**`WI_updateShowNextLoc`:** Decrements `cnt`, toggles `snl_pointeron` every 20 tics for the flashing pointer (`(cnt & 31) < 20`), and transitions to `NoState` when done.

**`WI_drawShowNextLoc`:** Slams the background, draws animated background elements, draws splat markers on all completed levels, optionally marks the secret exit, and draws the flashing "You Are Here" arrow on the next level.

---

#### `WI_drawNoState`

```c
void WI_drawNoState(void);
```

**Purpose:** Draws the world map with the "You Are Here" pointer permanently on (not flashing) during the brief `NoState` pause.

---

#### Deathmatch Stats

```c
void WI_initDeathmatchStats(void);
void WI_updateDeathmatchStats(void);
void WI_drawDeathmatchStats(void);
```

**Purpose:** Manages the deathmatch frag-matrix reveal sequence:
- `WI_initDeathmatchStats`: Sets state, zeros display counters.
- `WI_updateDeathmatchStats`: State machine with `dm_state` 1-4. Odd states are pauses; state 2 ticks all frag counts toward actual values, clamped to ±99. State 4 waits for player input before proceeding.
- `WI_drawDeathmatchStats`: Renders the kill matrix grid with player icons, per-pair frag counts, and total columns.

---

#### Netgame Stats

```c
void WI_initNetgameStats(void);
void WI_updateNetgameStats(void);
void WI_drawNetgameStats(void);
```

**Purpose:** Co-op netgame statistics display. Shows each player's kill/item/secret percentages in columns (and optionally frags if any player has kills). The reveal sequence uses `ng_state` 1-10: odd states are pauses, even states count up one category (2=kills, 4=items, 6=secrets, 8=frags).

---

#### Single-Player Stats

```c
void WI_initStats(void);
void WI_updateStats(void);
void WI_drawStats(void);
```

**Purpose:** Single-player end-of-level statistics. Reveals kill%, item%, secret%, time, and par time in sequence using `sp_state` 1-10. Statistics count up by 2% per tic (or 3 seconds per tic for time) until reaching their actual values. A pistol sound plays every 4 tics and an explosion sound marks each category's completion.

**Sentinel value:** Counts initialized to -1 are not yet displayed; `WI_drawPercent` and `WI_drawTime` interpret negative values as "do not draw".

---

#### `WI_checkForAccelerate`

```c
void WI_checkForAccelerate(void);
```

**Purpose:** Polls all active players' command buttons to detect fire or use button presses, setting `acceleratestage = 1` if either is detected. Uses edge-triggered detection (tracks `player->attackdown` / `player->usedown`) to avoid repeat-firing on held buttons.

---

### Main Public Interface

#### `WI_Ticker`

```c
void WI_Ticker(void);
```

**Purpose:** Per-tic update function. Called by the main game loop every tic while the intermission is active.

**Key logic:**
1. Increments `bcnt`.
2. On the very first tic (`bcnt == 1`), starts intermission music (`mus_dm2int` for DOOM II, `mus_inter` for episode DOOM).
3. Calls `WI_checkForAccelerate`.
4. Dispatches to the appropriate update function based on `state` and game mode.

---

#### `WI_Drawer`

```c
void WI_Drawer(void);
```

**Purpose:** Per-frame draw function. Called by the main render loop every frame while the intermission is active.

**Key logic:** Dispatches to `WI_drawDeathmatchStats`, `WI_drawNetgameStats`, `WI_drawStats`, `WI_drawShowNextLoc`, or `WI_drawNoState` based on `state` and game mode flags.

---

#### `WI_loadData`

```c
void WI_loadData(void);
```

**Purpose:** Loads all graphical assets used by the intermission screen from the WAD. Called once at the start of each intermission.

**Key logic:**
- Selects and loads the background map image.
- Loads level name patches: `CWILVxx` for commercial, `WILVxy` for episodes.
- For episodes, loads "You Are Here" arrows, splat, and all per-episode animation frames (`WIAepsodanimframe`).
- Loads all UI element patches: digit numerals, percent, colon, finished/entering banners, category labels, time/par/sucks, and multiplayer player icons.
- All patches loaded with `PU_STATIC` so they persist for the duration of the intermission.

---

#### `WI_unloadData`

```c
void WI_unloadData(void);
```

**Purpose:** Downgrades all intermission graphics from `PU_STATIC` to `PU_CACHE`, allowing the zone allocator to reclaim their memory when needed. Called by `WI_End`.

**Key logic:** `Z_ChangeTag(patch_ptr, PU_CACHE)` for every loaded patch. The `lnames` array itself is freed with `Z_Free`.

---

#### `WI_initVariables`

```c
void WI_initVariables(wbstartstruct_t* wbstartstruct);
```

**Purpose:** Initializes all intermission state variables from the provided `wbstartstruct_t`. Handles validation and edge cases.

**Key logic:**
- Stores `wbs`, `me`, `plrs`.
- Ensures `maxkills`, `maxitems`, `maxsecret` are at least 1 (avoids division by zero in percentage calculations).
- In non-retail mode, remaps episode 3 back to episode 0 (so Thy Flesh Consumed maps correctly use episode 0 resources).

---

#### `WI_Start`

```c
void WI_Start(wbstartstruct_t* wbstartstruct);
```

**Purpose:** The main entry point for starting an intermission screen. Called by the game engine when a level ends.

**Parameters:**
- `wbstartstruct` - Pointer to a fully populated `wbstartstruct_t` containing all level completion data.

**Key logic:** Calls `WI_initVariables`, `WI_loadData`, then dispatches to the appropriate stat initialization function based on game mode (`WI_initDeathmatchStats`, `WI_initNetgameStats`, or `WI_initStats`).

---

## Data Structures

### `animenum_t`

```c
typedef enum {
    ANIM_ALWAYS,
    ANIM_RANDOM,
    ANIM_LEVEL
} animenum_t;
```

**Purpose:** Specifies the behavior mode for a background animation on the world map:
- `ANIM_ALWAYS`: Cycles continuously throughout the intermission (e.g., lava bubbles, waterfalls).
- `ANIM_RANDOM`: Plays occasionally at random intervals (e.g., things popping up).
- `ANIM_LEVEL`: Plays only when the next level is a specific map (e.g., reveals a level's icon once it's been beaten).

---

### `point_t`

```c
typedef struct {
    int x;
    int y;
} point_t;
```

**Purpose:** A simple 2D pixel coordinate. Used in `lnodes` for level positions and in `anim_t` for animation positions on the world map.

---

### `anim_t`

```c
typedef struct {
    animenum_t type;    // Animation behavior type
    int        period;  // Tics between frames
    int        nanims;  // Number of frames
    point_t    loc;     // Screen position of the animation

    int        data1;   // ALWAYS: unused; RANDOM: period deviation (<256); LEVEL: target level number
    int        data2;   // ALWAYS: unused; RANDOM: base period; LEVEL: unused

    patch_t*   p[3];    // Up to 3 frame patches

    // Runtime state (must be zero-initialized before use):
    int        nexttic;   // bcnt value at which to advance to the next frame
    int        lastdrawn; // Last drawn frame index (unused in practice)
    int        ctr;       // Current frame index (-1 = not yet started)
    int        state;     // State for RANDOM/LEVEL animation logic
} anim_t;
```

**Purpose:** Describes one animated element on the world map background. Static arrays of these are defined for each episode (`epsd0animinfo`, `epsd1animinfo`, `epsd2animinfo`). The runtime fields are initialized per-intermission by `WI_initAnimatedBack`.

---

## Dependencies

| File | Reason |
|------|--------|
| `z_zone.h` | `Z_Malloc`, `Z_Free`, `Z_ChangeTag` for intermission graphics memory management. |
| `m_random.h` | `M_Random` for randomizing animation timings. |
| `m_swap.h` | `SHORT` byte-swap macro for reading patch dimensions. |
| `i_system.h` | `I_Error` for fatal errors. |
| `w_wad.h` | `W_CacheLumpName` for loading all intermission graphics from the WAD. |
| `g_game.h` | `G_WorldDone` to signal the end of intermission and start the next level. |
| `r_local.h` | Provides `patch_t` and rendering constants. |
| `s_sound.h` | `S_StartSound`, `S_ChangeMusic` for intermission sound effects and music. |
| `doomstat.h` | Global game state variables: `gamemode`, `deathmatch`, `netgame`, `playeringame[]`, `players[]`, `french`. |
| `sounds.h` | Sound and music lump name constants (`sfx_pistol`, `sfx_barexp`, `mus_inter`, etc.). |
| `v_video.h` | `V_DrawPatch`, `V_MarkRect` for drawing to the screen buffer. |
| `wi_stuff.h` | Self-header; provides `stateenum_t` and the public function declarations. |
| `doomdef.h` | `SCREENWIDTH`, `SCREENHEIGHT`, `TICRATE`, `MAXPLAYERS`, and game mode constants. |
