# File Overview

`p_tick.h` is the public header for the thinker/ticker subsystem declared in `p_tick.c`. It exposes the single entry point `P_Ticker`, which is the top-level per-game-tic driver for all play-simulation logic (player movement, monster AI, sector specials, etc.).

The header is intentionally minimal — the thinker list management functions (`P_InitThinkers`, `P_AddThinker`, `P_RemoveThinker`) are declared in `p_local.h` rather than here, because they are internal play-module utilities rather than cross-subsystem interfaces. Only the top-level ticker needs to be visible to the game loop.

## Global Variables

This header declares no global variables.

## Functions

### `P_Ticker`

```c
void P_Ticker(void);
```

**Purpose:** Declared here for use by the game controller (historically `C_Ticker`, now integrated into `G_Ticker`/`D_DoomLoop`). Drives one game tic of all play simulation: player thinking, thinker execution, sector special updates, item respawning, and level-time increment.

See `p_tick.c` for the full implementation details.

**Parameters:** None.

**Return Value:** `void`

**Notes from header comment:** "Called by C_Ticker, can call G_PlayerExited. Carries out all thinking of monsters and players." The reference to `G_PlayerExited` indicates that player death/exit detection can be triggered as a side effect of `P_Ticker` through `P_PlayerThink`.

## Data Structures

No data structures are defined in this header.

## Dependencies

| File | Reason |
|---|---|
| (none — only include guard) | The header is self-contained; it relies on the compiler having already seen `doomtype.h` or `doomdef.h` for `void`, but does not explicitly include them. |
