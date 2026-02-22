# File Overview

**Source file:** `linuxdoom-1.10/f_finale.h`

`f_finale.h` is the public interface header for the finale/end-game animation subsystem implemented in `f_finale.c`. It declares the four functions that the main game loop calls to drive the end-of-game sequence, and is guarded by the include sentinel `__F_FINALE__`.

The header is intentionally minimal: all state variables and internal helpers (`F_StartCast`, `F_CastTicker`, `F_CastResponder`, `F_CastDrawer`, `castorder[]`, etc.) are private to `f_finale.c` and not exposed here.

---

## Global Variables

None. This header declares no variables; all finale state is file-private in `f_finale.c`.

---

## Functions

### `F_Responder`

```c
boolean F_Responder(event_t* ev);
```

**Purpose:** Routes player input events to the finale system. During the cast parade (stage 2) it forwards key-down events to `F_CastResponder` so the player can kill the displayed monster. In stages 0 and 1 it always returns `false`.

**Called by:** The main event-dispatch loop in `G_Responder` (`g_game.c`) when `gamestate == GS_FINALE`.

**Parameters:**
- `ev` — pointer to the current input event

**Returns:** `true` if the event was consumed by the finale handler, `false` otherwise.

---

### `F_Ticker`

```c
void F_Ticker(void);
```

**Purpose:** Advances the finale animation state by one game tic. Handles stage transitions (text → art screen, or initiating the cast parade on map 30), monitors player button input in DOOM II to allow skipping the text stage, and delegates tic logic to `F_CastTicker` during stage 2.

**Called by:** `G_Ticker` (`g_game.c`) each tic when `gamestate == GS_FINALE`.

**Parameters:** none
**Returns:** void

---

### `F_Drawer`

```c
void F_Drawer(void);
```

**Purpose:** Renders the current frame of the finale to the primary screen buffer. Dispatches to the appropriate renderer based on `finalestage`:
- Stage 0: `F_TextWrite` (tiled flat + proportional text crawl)
- Stage 1: episode-specific art (`CREDIT`, `HELP2`, `VICTORY2`, bunny scroll, `ENDPIC`)
- Stage 2: `F_CastDrawer` (BOSSBACK + cast member sprite + name label)

**Called by:** The main render loop each display frame when `gamestate == GS_FINALE`.

**Parameters:** none
**Returns:** void

---

### `F_StartFinale`

```c
void F_StartFinale(void);
```

**Purpose:** Initializes and starts the end-game finale sequence. Sets `gamestate = GS_FINALE`, selects the appropriate background flat and text string based on `gamemode` and `gameepisode`/`gamemap`, starts the finale music, and resets `finalestage` and `finalecount` to zero.

**Called by:** `G_Ticker` in `g_game.c` when `gameaction == ga_victory`, and directly from `G_WorldDone` for DOOM II inter-boss maps.

**Parameters:** none
**Returns:** void

---

## Data Structures

None defined in this header. The `castinfo_t` struct is defined privately in `f_finale.c`.

---

## Dependencies

| Header | Reason |
|--------|--------|
| `doomtype.h` | Provides the `boolean` typedef used in `F_Responder`'s return type |
| `d_event.h` | Provides the `event_t` type used in `F_Responder`'s parameter |
