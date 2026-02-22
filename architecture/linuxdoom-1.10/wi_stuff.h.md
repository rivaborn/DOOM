# File Overview

**Source file:** `linuxdoom-1.10/wi_stuff.h`
**Module:** Intermission Screen Interface (`WI_` prefix)

`wi_stuff.h` is the public interface header for DOOM's intermission (between-level) screen system. It declares the three entry points that the main game loop uses to drive the intermission: one to start it with level statistics, one to advance its animation each game tick, and one to render it to the screen buffer. The header also defines the `stateenum_t` enumeration that describes which phase of the intermission is currently active.

The intermission screen appears after each level is completed. It shows per-player kill, item, and secret percentages, elapsed time, par time, and (in single-player) animates the world map to indicate the next level. In deathmatch it tallies frag counts across all players.

The implementation lives in `wi_stuff.c`. This header is included by `g_game.c` (which calls `WI_Start` to hand off level stats) and by the main loop in `d_main.c` (which calls `WI_Ticker` and `WI_Drawer` every tic while the intermission is active).

---

## Data Structures

### `stateenum_t`

```c
typedef enum
{
    NoState    = -1,
    StatCount,       // 0 - counting up kills/items/secrets/time
    ShowNextLoc      // 1 - showing the next level on the world map
} stateenum_t;
```

Represents the two visible phases of the intermission sequence, plus a sentinel value used during state transitions:

| Enumerator   | Value | Meaning |
|--------------|-------|---------|
| `NoState`    | `-1`  | Transition/uninitialized sentinel. Used internally in `wi_stuff.c` to force a state change on the very first ticker call after `WI_Start`. |
| `StatCount`  | `0`   | The "count-up" phase where kill percentage, item percentage, secret percentage, and level time tick upward toward the actual totals. This is the first thing the player sees. |
| `ShowNextLoc`| `1`   | The world-map phase (Episode 1/2/3 only) where an animation highlights the completed level and then points to the next one. Skipped for DOOM II's single-episode structure. |

---

## Functions

### `WI_Ticker`

```c
void WI_Ticker(void);
```

**Purpose:** Advances intermission logic by one game tic (1/35 second). Called once per tic by the main game loop while `gamestate == GS_INTERMISSION`.

**Parameters:** None.

**Return value:** None.

**Key logic:**
- Decrements the accelerate-counter so that pressing a key or mouse button speeds up the count-up animations.
- Drives the state machine: in `StatCount` mode it increments displayed kill/item/secret/time counters toward their targets at a fixed rate per tic; transitions to `ShowNextLoc` once all counts reach their targets.
- In `ShowNextLoc` mode it advances the world-map "you are here" animation frame and eventually transitions to ending the intermission (returning control to `G_WorldDone`).
- Handles multiplayer-specific logic: in cooperative mode it counts up statistics for each player in sequence; in deathmatch mode it fills in the frag matrix.

---

### `WI_Drawer`

```c
void WI_Drawer(void);
```

**Purpose:** Renders the current intermission frame directly into the linear screen buffer (`screens[0]`). Called once per display frame (same cadence as `WI_Ticker` in the original engine, since they share the same game loop iteration).

**Parameters:** None.

**Return value:** None.

**Key logic:**
- Draws the full-screen background flat/patch appropriate to the current episode.
- Overlays the level title and "Finished" / "Entering" banners.
- Depending on `state`:
  - `StatCount`: draws partially-complete percentage bars or count numbers.
  - `ShowNextLoc`: draws the world map with animated "splat" markers for completed levels and a flashing "you are here" marker for the next level.
- In multiplayer, draws a grid of player-colored statistics.
- Uses `V_DrawPatch` for all sprite-based drawing.

---

### `WI_Start`

```c
void WI_Start(wbstartstruct_t* wbstartstruct);
```

**Purpose:** Initializes the intermission system for a new between-level sequence. Called by `G_DoWorldDone` / `G_DoCompleted` in `g_game.c` immediately after a level ends.

**Parameters:**

| Parameter      | Type                | Description |
|----------------|---------------------|-------------|
| `wbstartstruct`| `wbstartstruct_t *` | Pointer to a structure filled in by the game logic with all statistics and context for the completed level (see Data Structures below). |

**Return value:** None.

**Key logic:**
- Saves the pointer to the stats structure for use by `WI_Ticker` and `WI_Drawer`.
- Calls `WI_LoadData` to load all patches (background, percent sign, colon, digit fonts, animation frames) into the zone heap with `PU_STATIC` tag.
- Resets all internal counters (displayed kills, items, secrets, time) to zero so the count-up animation starts fresh.
- Sets `state = NoState` and calls `WI_InitNoState` so the very first ticker invocation will transition immediately into `StatCount`.
- Starts the intermission music track (`mus_inter` / `mus_dm2int`).

---

## Dependencies

| Header / Module | Reason |
|-----------------|--------|
| `doomdef.h`     | Provides `boolean`, `byte`, `MAXPLAYERS`, `gamemode` enum, and other engine-wide fundamental types and constants. |
| `d_player.h`    | Defines `wbstartstruct_t` and `wbplayerstruct_t` (the structures passed to `WI_Start`; see below). |
| `v_video.h`     | (Commented out in the header, but included by `wi_stuff.c`) Provides `V_DrawPatch` and the `screens[]` framebuffer array used by `WI_Drawer`. |

### Referenced external structures

#### `wbplayerstruct_t` (defined in `d_player.h`)

```c
typedef struct
{
    boolean in;       // whether this player slot is active in game
    int     skills;   // kills scored this level
    int     sitems;   // items collected this level
    int     ssecret;  // secrets found this level
    int     stime;    // level time in tics
    int     frags[4]; // frags against each other player (deathmatch)
    int     score;    // running score (deathmatch)
} wbplayerstruct_t;
```

#### `wbstartstruct_t` (defined in `d_player.h`)

```c
typedef struct
{
    int              epsd;      // episode number (0-2)
    boolean          didsecret; // true if the secret level was visited
    int              last;      // index of the level just completed (0-based)
    int              next;      // index of the next level (0-based)
    int              maxkills;  // total killable monsters on the level
    int              maxitems;  // total items on the level
    int              maxsecret; // total secrets on the level
    int              maxfrags;  // (unused placeholder)
    int              partime;   // par time in tics
    int              pnum;      // index of the local player in game
    wbplayerstruct_t plyr[MAXPLAYERS]; // per-player statistics
} wbstartstruct_t;
```

This is the primary data package handed from the game logic to the intermission system. It is stored in the global `wminfo` variable (`doomstat.h`) and its address is passed to `WI_Start`.
