# File Overview

**Source file:** `linuxdoom-1.10/g_game.h`

`g_game.h` is the public interface header for the core game logic module (`g_game.c`). It declares all functions that other subsystems use to interact with the game state machine: starting new games, loading and saving games, recording and playing back demos, responding to input events, advancing the game one tic, and triggering level transitions.

The header deliberately exposes no variables — all game state (`gameaction`, `gamestate`, `players[]`, etc.) is declared directly in `g_game.c` or in `doomstat.h` and accessed by other modules via externs there.

---

## Global Variables

None declared in this header. All game-state globals are exposed through `doomstat.h`.

---

## Functions

### `G_DeathMatchSpawnPlayer`

```c
void G_DeathMatchSpawnPlayer(int playernum);
```

**Purpose:** Spawns the specified player at a random deathmatch start spot, checking that the spot is unoccupied. Falls back to the regular player start if no valid deathmatch spot is found after 20 attempts. Requires at least 4 deathmatch spots in the level or calls `I_Error`.

**Parameters:**
- `playernum` — zero-based player index (0–3)

---

### `G_InitNew`

```c
void G_InitNew(skill_t skill, int episode, int map);
```

**Purpose:** Immediately initializes a new game with the given parameters. Clamps skill, episode, and map to valid ranges for the current `gamemode`. Sets up nightmare/fast monster speed adjustments, initializes all player states to `PST_REBORN`, selects the sky texture, and calls `G_DoLoadLevel` to load the first map.

**Called by:** `G_DoNewGame`, `G_DoLoadGame`, `G_DoPlayDemo`, and startup code.

**Parameters:**
- `skill` — difficulty level (`sk_baby` through `sk_nightmare`)
- `episode` — episode number (1–4 for DOOM 1; 1 for DOOM II)
- `map` — map number within the episode (1–9 for DOOM 1; 1–32 for DOOM II)

---

### `G_DeferedInitNew`

```c
void G_DeferedInitNew(skill_t skill, int episode, int map);
```

**Purpose:** Schedules a new game to begin at the next tic by storing the parameters and setting `gameaction = ga_newgame`. Safer than calling `G_InitNew` directly from menu or startup code because the actual game-state change is deferred to `G_Ticker`.

**Parameters:** same as `G_InitNew`.

---

### `G_DeferedPlayDemo`

```c
void G_DeferedPlayDemo(char* demo);
```

**Purpose:** Schedules demo playback to begin at the next tic. Stores the demo lump name and sets `gameaction = ga_playdemo`.

**Parameters:**
- `demo` — WAD lump name of the demo to play (e.g., `"DEMO1"`)

---

### `G_LoadGame`

```c
void G_LoadGame(char* name);
```

**Purpose:** Schedules a savegame to be loaded at the next tic. Copies the file path into `savename` and sets `gameaction = ga_loadgame`. Safe to call from the menu.

**Parameters:**
- `name` — filesystem path to the `.dsg` savegame file

---

### `G_DoLoadGame`

```c
void G_DoLoadGame(void);
```

**Purpose:** Executes the deferred savegame load. Reads the file, validates the version string, restores skill/episode/map and player states, calls `G_InitNew` to set up the level, then dearchives all game objects via the `P_UnArchive*` functions. Calls `I_Error` if the savegame trailer byte is not `0x1d`.

**Called by:** `G_Ticker` when `gameaction == ga_loadgame`.

---

### `G_SaveGame`

```c
void G_SaveGame(int slot, char* description);
```

**Purpose:** Schedules a savegame write at the next tic. Stores the slot index and description string, then sets `sendsave = true` so that `G_BuildTiccmd` will embed the save request in the next ticcmd (ensuring all players see it simultaneously in a network game).

**Parameters:**
- `slot` — savegame slot index (0–7, used to form the filename `doomsav{slot}.dsg`)
- `description` — 24-byte user-visible description string

---

### `G_RecordDemo`

```c
void G_RecordDemo(char* name);
```

**Purpose:** Initializes demo recording. Allocates a demo buffer (default 128 KB, overridable with `-maxdemo`), appends `.lmp` to the name, and sets `demorecording = true`. Recording actually begins when `G_BeginRecording` is called.

**Parameters:**
- `name` — base filename for the demo (without extension)

---

### `G_BeginRecording`

```c
void G_BeginRecording(void);
```

**Purpose:** Writes the demo header into the demo buffer. The header records VERSION, skill, episode, map, deathmatch flag, respawn/fast/nomonsters parameters, consoleplayer, and the `playeringame[]` array. Must be called after `G_InitNew` has set up game parameters.

---

### `G_PlayDemo`

```c
void G_PlayDemo(char* name);
```

**Purpose:** Synonym / startup path for demo playback; declared here but the core implementation is `G_DoPlayDemo`. Typically this is `G_DeferedPlayDemo` or a direct call to the internal path from startup code. (In practice, `G_DeferedPlayDemo` is the one used at runtime.)

**Parameters:**
- `name` — WAD lump name of the demo

---

### `G_TimeDemo`

```c
void G_TimeDemo(char* name);
```

**Purpose:** Starts a timing benchmark demo. Sets `timingdemo = true` and `singletics = true` (no frame-rate limiting), then schedules the demo for playback. When the demo ends, `G_CheckDemoStatus` calls `I_Error` to print the benchmark result.

**Parameters:**
- `name` — WAD lump name of the demo to time

---

### `G_CheckDemoStatus`

```c
boolean G_CheckDemoStatus(void);
```

**Purpose:** Called at the end of a demo (natural completion or player key press) or on death/level-exit during recording.

- **Timing mode:** Computes elapsed real time and calls `I_Error` to report tics/realtics.
- **Playback mode:** Releases the demo buffer, resets all demo/net/game flags, calls `D_AdvanceDemo` to cycle to the next demo, returns `true`.
- **Recording mode:** Writes the `DEMOMARKER` (0x80), flushes the buffer to disk, frees the buffer, calls `I_Error` with "Demo recorded" message.

**Returns:** `true` if a demo-loop action was scheduled (so the caller should return immediately), `false` otherwise.

---

### `G_ExitLevel`

```c
void G_ExitLevel(void);
```

**Purpose:** Triggers a normal (non-secret) level exit. Sets `secretexit = false` and `gameaction = ga_completed`.

---

### `G_SecretExitLevel`

```c
void G_SecretExitLevel(void);
```

**Purpose:** Triggers a secret level exit. Sets `secretexit = true` unless in the German DOOM II edition without Wolf3D levels (map31 not present), in which case falls back to a normal exit. Sets `gameaction = ga_completed`.

---

### `G_WorldDone`

```c
void G_WorldDone(void);
```

**Purpose:** Called by the intermission screen (`WI_Ticker`) when the player has finished viewing the results. Sets `gameaction = ga_worlddone`. In DOOM II, also starts the finale sequence for boss maps (6, 11, 20, 30, and secret exits from 15 and 31).

---

### `G_Ticker`

```c
void G_Ticker(void);
```

**Purpose:** The master per-tic update function. Called once per game tic by `D_DoomLoop`. Performs:
1. Player reborn checks.
2. `gameaction` dispatch (loads levels, saves, demos, level completion, victory, screenshots).
3. Ticcmd distribution from `netcmds[]` to players, with demo read/write.
4. Turbo-cheat detection (broadcasts a HUD message if `forwardmove > TURBOTHRESHOLD`).
5. Network consistency checks.
6. Special button handling (pause, savegame from ticcmd).
7. Per-gamestate tick dispatching: `P_Ticker` / `ST_Ticker` / `AM_Ticker` / `HU_Ticker` (level), `WI_Ticker` (intermission), `F_Ticker` (finale), `D_PageTicker` (demo screen).

---

### `G_Responder`

```c
boolean G_Responder(event_t* ev);
```

**Purpose:** The master input event handler. Processes events before dispatching to subsystems. Handles:
- F12 spy-mode switching.
- Demo/title-screen key presses that open the menu.
- Level-active routing to `HU_Responder`, `ST_Responder`, `AM_Responder`.
- Finale routing to `F_Responder`.
- Raw keyboard up/down → `gamekeydown[]`.
- Mouse button state and X/Y deltas (scaled by `mouseSensitivity`).
- Joystick button state and axis values.
- Pause key → `sendpause`.

**Returns:** `true` if the event was consumed.

---

### `G_ScreenShot`

```c
void G_ScreenShot(void);
```

**Purpose:** Schedules a screenshot to be taken at the next tic by setting `gameaction = ga_screenshot`. The actual capture is done by `M_ScreenShot` in `G_Ticker`.

---

## Data Structures

None defined in this header.

---

## Dependencies

| Header | Reason |
|--------|--------|
| `doomdef.h` | `skill_t` enum, `boolean`, basic DOOM types |
| `d_event.h` | `event_t` type used by `G_Responder` |
