# File Overview

`st_stuff.h` is the public header for the DOOM status bar system. The status bar is the 32-pixel tall strip at the bottom of the screen that displays health, ammunition, armor, weapon selection, face expressions, and other critical player information during gameplay. This header defines the interface that the rest of the engine uses to interact with the status bar module (`st_stuff.c`).

The status bar is tightly coupled to the HUD (heads-up display) and responds to game events, updating every game tic. It also manages the classic "face" indicator that responds to damage direction and player health, as well as palette flashes for pain (red), pickup (golden), and berserk (red tint).

---

## Global Variables

This header does not declare any global variables directly. All state is internal to `st_stuff.c`. The status bar dimensions are exposed as macros rather than variables.

---

## Functions

### `ST_Responder`

```c
boolean ST_Responder(event_t* ev);
```

**Purpose:** Processes input events relevant to the status bar. In practice this handles keyboard shortcuts for weapon selection and cheats that are entered via the keyboard while the status bar is active.

**Parameters:**
- `ev` - Pointer to an `event_t` describing the input event (key press/release, mouse button, joystick).

**Return value:** `boolean` - Returns `true` if the event was consumed by the status bar and should not be passed to other subsystems; `false` if the event should continue propagating.

**Key logic:** The function is declared twice in this header (lines 42 and 77), which is a minor artifact of the original source. The effective declaration is the one on line 77.

---

### `ST_Ticker`

```c
void ST_Ticker(void);
```

**Purpose:** Updates all status bar state for one game tic. This is called every tic from the main game loop and is responsible for animating the face indicator, tracking ammo changes, updating the kill/item/secret counters in the automap state, and handling all time-based status bar transitions.

**Parameters:** None.

**Return value:** None.

**Key logic:** Drives the face animation state machine (which face expression to show based on recent damage, kills, and health), interpolates the displayed values toward their targets, and checks whether the status bar needs to be redrawn.

---

### `ST_Drawer`

```c
void ST_Drawer(boolean fullscreen, boolean refresh);
```

**Purpose:** Renders the status bar to the screen buffer. Called by the main render loop every frame.

**Parameters:**
- `fullscreen` - If `true`, the game is running in full-screen mode and the status bar is drawn as an overlay; if `false`, the status bar occupies its dedicated strip at the bottom.
- `refresh` - If `true`, forces a complete redraw of the status bar regardless of what has changed. If `false`, only dirty regions are redrawn.

**Return value:** None.

**Key logic:** Delegates to `st_lib.c` widget draw calls to render numbers (health, ammo, armor), the face graphic, the weapon indicator arms display, and the keys. Respects the `fullscreen` flag to decide the rendering mode.

---

### `ST_Start`

```c
void ST_Start(void);
```

**Purpose:** Initializes or resets the status bar state when the console player spawns on a level. Called at the beginning of each new map or respawn.

**Parameters:** None.

**Return value:** None.

**Key logic:** Resets all cached weapon/ammo/health values so the status bar will be fully redrawn on the first frame. Resets the face animation state and clears any pending palette indicators.

---

### `ST_Init`

```c
void ST_Init(void);
```

**Purpose:** One-time initialization of the status bar system at engine startup. Called from the startup sequence before the first level loads.

**Parameters:** None.

**Return value:** None.

**Key logic:** Loads all status bar graphics from the WAD (face sprites, number graphics, key icons, weapon arm images) using `W_CacheLumpName`. Initializes the `st_lib` widget system and registers all the numeric display widgets for health, ammunition counts, and armor.

---

## Data Structures

### `st_stateenum_t`

```c
typedef enum
{
    AutomapState,
    FirstPersonState
} st_stateenum_t;
```

**Purpose:** Distinguishes between the two display modes of the status bar. In `AutomapState` the status bar is displayed alongside the automap. In `FirstPersonState` it is displayed during normal first-person gameplay. Some widgets are only drawn in one mode or the other.

---

### `st_chatstateenum_t`

```c
typedef enum
{
    StartChatState,
    WaitDestState,
    GetChatState
} st_chatstateenum_t;
```

**Purpose:** Tracks the state of the multiplayer chat input system that runs through the status bar. The three states represent:
- `StartChatState` - Initial state, waiting for the player to initiate a chat message.
- `WaitDestState` - Waiting for the player to select a destination (in multiplayer: specific player or all).
- `GetChatState` - Actively receiving typed characters for the chat message.

---

## Dependencies

| File | Reason |
|------|--------|
| `doomtype.h` | Provides the `boolean` type used in function signatures. |
| `d_event.h` | Provides the `event_t` type consumed by `ST_Responder`. |
| `doomdef.h` | Implicitly required for `SCREEN_MUL`, `SCREENWIDTH`, `SCREENHEIGHT` used in the dimension macros. |

The macros defined in this header depend on constants from `doomdef.h`:

| Macro | Value | Meaning |
|-------|-------|---------|
| `ST_HEIGHT` | `32 * SCREEN_MUL` | Height of the status bar in pixels (32 at 1x scaling). |
| `ST_WIDTH` | `SCREENWIDTH` | Width of the status bar; matches full screen width (320 pixels at standard resolution). |
| `ST_Y` | `SCREENHEIGHT - ST_HEIGHT` | Y-coordinate of the top of the status bar (168 at standard 200-pixel height). |
