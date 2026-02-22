# File Overview

**Source file:** `linuxdoom-1.10/g_game.c`

`g_game.c` is the central game logic file in DOOM. It owns the **game state machine**, **input processing**, **tic command construction**, **player lifecycle management**, **save/load**, and **demo record/playback**. Nearly every other subsystem in the engine either drives its behavior through variables declared here or is ticked by functions called from here.

### Responsibilities at a glance

| Area | Key functions |
|------|--------------|
| Game state machine | `G_Ticker`, `G_Responder` |
| Level loading | `G_DoLoadLevel`, `G_InitNew`, `G_DoNewGame` |
| Input → ticcmd | `G_BuildTiccmd` |
| Save / load | `G_SaveGame`, `G_DoSaveGame`, `G_LoadGame`, `G_DoLoadGame` |
| Demo record | `G_RecordDemo`, `G_BeginRecording`, `G_WriteDemoTiccmd`, `G_CheckDemoStatus` |
| Demo playback | `G_DeferedPlayDemo`, `G_DoPlayDemo`, `G_ReadDemoTiccmd`, `G_TimeDemo` |
| Level completion | `G_ExitLevel`, `G_SecretExitLevel`, `G_DoCompleted`, `G_WorldDone`, `G_DoWorldDone` |
| Player lifecycle | `G_InitPlayer`, `G_PlayerReborn`, `G_PlayerFinishLevel`, `G_DoReborn`, `G_CheckSpot`, `G_DeathMatchSpawnPlayer` |

---

## Global Variables

### Game state

| Type | Name | Purpose |
|------|------|---------|
| `gameaction_t` | `gameaction` | Pending action for the state machine to execute this tic (e.g., `ga_loadlevel`, `ga_savegame`, `ga_completed`, `ga_victory`, `ga_worlddone`, `ga_screenshot`, `ga_nothing`) |
| `gamestate_t` | `gamestate` | Current high-level state: `GS_LEVEL`, `GS_INTERMISSION`, `GS_FINALE`, `GS_DEMOSCREEN` |
| `skill_t` | `gameskill` | Skill level of the current game |
| `boolean` | `respawnmonsters` | True on Nightmare or with `-respawn`; causes dead monsters to respawn |
| `int` | `gameepisode` | Current episode (1–4 for DOOM 1; always 1 for DOOM II) |
| `int` | `gamemap` | Current map number within the episode |
| `boolean` | `paused` | True when the game is paused |
| `boolean` | `sendpause` | Set to true to inject a pause ticcmd next tic |
| `boolean` | `sendsave` | Set to true to inject a savegame ticcmd next tic |
| `boolean` | `usergame` | True when the player is in an active (saveable) game, not watching a demo |
| `boolean` | `viewactive` | True when the 3D view renderer should be active |

### Timing / benchmarking

| Type | Name | Purpose |
|------|------|---------|
| `boolean` | `timingdemo` | True during a timed benchmark demo run |
| `boolean` | `nodrawers` | If true, skip all rendering (for pure timing tests) |
| `boolean` | `noblit` | If true, skip screen blit (for comparative timing) |
| `int` | `starttime` | `I_GetTime()` value at level start, used for timing demo calculation |

### Multiplayer / network

| Type | Name | Purpose |
|------|------|---------|
| `boolean` | `deathmatch` | True if this is a deathmatch session |
| `boolean` | `netgame` | True when playing over a network (or simulating one with a netdemo) |
| `boolean` | `playeringame[MAXPLAYERS]` | Slot-active flags for each of the 4 possible players |
| `player_t` | `players[MAXPLAYERS]` | Full player state structs |
| `int` | `consoleplayer` | Index of the local player receiving input and displaying messages |
| `int` | `displayplayer` | Index of the player whose viewpoint is rendered (can differ in spy mode) |
| `int` | `gametic` | Global tic counter since the game started |
| `int` | `levelstarttic` | `gametic` value at the moment the current level was loaded |
| `int` | `totalkills` | Total monster count in the level (for the intermission percentage) |
| `int` | `totalitems` | Total item count in the level |
| `int` | `totalsecret` | Total secret count in the level |
| `short` | `consistancy[MAXPLAYERS][BACKUPTICS]` | Network consistency check values; compared against ticcmd's `consistancy` field |

### Demo system

| Type | Name | Purpose |
|------|------|---------|
| `char` | `demoname[32]` | Filename for the demo being recorded |
| `boolean` | `demorecording` | True while recording a demo |
| `boolean` | `demoplayback` | True while playing back a demo |
| `boolean` | `netdemo` | True when playing back a multi-player demo |
| `byte*` | `demobuffer` | Base pointer of the demo data buffer |
| `byte*` | `demo_p` | Current read/write cursor within `demobuffer` |
| `byte*` | `demoend` | One-past-end of `demobuffer`; used to detect buffer overflow during recording |
| `boolean` | `singledemo` | If true, quit after the command-line demo finishes playing |

### Save/load

| Type | Name | Purpose |
|------|------|---------|
| `char` | `savename[256]` | Filesystem path to the savegame file to load |
| `byte*` | `savebuffer` | Pointer to the savegame memory buffer (borrowed from `screens[1]+0x4000` during save) |
| `int` | `savegameslot` | Slot index (0–7) of the pending save/load operation |
| `char` | `savedescription[32]` | User-visible description string for the current save slot |

### Deferred new-game parameters

| Type | Name | Purpose |
|------|------|---------|
| `skill_t` | `d_skill` | Skill stored by `G_DeferedInitNew` |
| `int` | `d_episode` | Episode stored by `G_DeferedInitNew` |
| `int` | `d_map` | Map stored by `G_DeferedInitNew` |

### Intermission

| Type | Name | Purpose |
|------|------|---------|
| `wbstartstruct_t` | `wminfo` | Fully populated intermission parameter struct passed to `WI_Start` |

### Input — keyboard

| Type | Name | Purpose |
|------|------|---------|
| `int` | `key_right`, `key_left` | Keycodes for turning right/left |
| `int` | `key_up`, `key_down` | Keycodes for moving forward/backward |
| `int` | `key_strafeleft`, `key_straferight` | Keycodes for lateral strafing |
| `int` | `key_fire`, `key_use`, `key_strafe`, `key_speed` | Keycodes for fire, use, strafe-modifier, run |
| `boolean` | `gamekeydown[NUMKEYS]` | Per-keycode down state; indexed by keycode (0–255) |
| `int` | `turnheld` | Tic counter for held turn keys; enables accelerative turning |

### Input — mouse

| Type | Name | Purpose |
|------|------|---------|
| `int` | `mousebfire`, `mousebstrafe`, `mousebforward` | Mouse button indices for fire, strafe, forward |
| `boolean` | `mousearray[4]` | Raw mouse button storage; `mousebuttons` points to `&mousearray[1]` to allow index [-1] |
| `boolean*` | `mousebuttons` | Mouse button state array (allows [-1] indexing for safety) |
| `int` | `mousex`, `mousey` | Mouse delta values for the current tic, consumed once per `G_BuildTiccmd` call |
| `int` | `dclicktime`, `dclickstate`, `dclicks` | Double-click detection for forward mouse button (triggers `BT_USE`) |
| `int` | `dclicktime2`, `dclickstate2`, `dclicks2` | Double-click detection for strafe mouse button (triggers `BT_USE`) |

### Input — joystick

| Type | Name | Purpose |
|------|------|---------|
| `int` | `joybfire`, `joybstrafe`, `joybuse`, `joybspeed` | Joystick button indices for fire, strafe, use, speed |
| `int` | `joyxmove`, `joyymove` | Joystick axis values; repeated (not cleared after use) |
| `boolean` | `joyarray[5]` | Raw joystick button storage; `joybuttons = &joyarray[1]` |
| `boolean*` | `joybuttons` | Joystick button state array ([-1] indexable) |

### Movement constants

| Type | Name | Purpose |
|------|------|---------|
| `fixed_t` | `forwardmove[2]` | Forward speed for walk (`0x19`) and run (`0x32`) |
| `fixed_t` | `sidemove[2]` | Strafe speed for walk (`0x18`) and run (`0x28`) |
| `fixed_t` | `angleturn[3]` | Turn rates: normal (`640`), fast (`1280`), slow (`320`) |

### Body queue

| Type | Name | Purpose |
|------|------|---------|
| `mobj_t*` | `bodyque[BODYQUESIZE]` | Ring buffer of dead player corpse objects; oldest is freed when a new respawn occurs (size = 32) |
| `int` | `bodyqueslot` | Next write index into `bodyque`; also used modulo `BODYQUESIZE` for wraparound |

### Statistics

| Type | Name | Purpose |
|------|------|---------|
| `void*` | `statcopy` | If non-null, `wminfo` is copied here at level completion (for an external statistics driver) |

### Par time tables

| Type | Name | Purpose |
|------|------|---------|
| `int` | `pars[4][10]` | Par times (in seconds) for DOOM 1 maps, indexed `[episode][map]`. Row 0 is unused padding. |
| `int` | `cpars[32]` | Par times for DOOM II maps 1–32. |

### Compile-time constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `SAVEGAMESIZE` | `0x2c000` (180 224) | Maximum savegame buffer size in bytes |
| `SAVESTRINGSIZE` | 24 | Length of the user description field in a savegame |
| `VERSIONSIZE` | 16 | Length of the version string field in a savegame |
| `MAXPLMOVE` | `forwardmove[1]` | Maximum per-tic forward/side movement units |
| `TURBOTHRESHOLD` | `0x32` (50) | Forward-move value above which a "turbo" warning is broadcast |
| `SLOWTURNTICS` | 6 | Number of held-turn tics before switching from slow to fast turn speed |
| `NUMKEYS` | 256 | Size of `gamekeydown` array |
| `BODYQUESIZE` | 32 | Number of player corpses to keep before freeing the oldest |
| `DEMOMARKER` | `0x80` | Byte value appended to a demo stream to signal end-of-demo |

---

## Functions

### `G_CmdChecksum`

```c
int G_CmdChecksum(ticcmd_t* cmd)
```

**Purpose:** Computes a simple checksum over the fields of a `ticcmd_t` (all except the last `int`-sized field). Used for debugging; not called in release-mode paths.

**Parameters:** `cmd` — pointer to the tic command

**Returns:** Sum of all `int`-sized words in the command except the last.

---

### `G_BuildTiccmd`

```c
void G_BuildTiccmd(ticcmd_t* cmd)
```

**Purpose:** Constructs the local player's tic command for the current tic by aggregating all input sources: keyboard, mouse, and joystick. If recording a demo, the command is also written to the demo buffer via `G_WriteDemoTiccmd`.

**Key logic:**

1. **Base command:** Calls `I_BaseTiccmd()` (returns a zeroed or externally-driven command) and copies it into `cmd`.

2. **Consistency field:** Sets `cmd->consistancy` from the local player's consistency history slot.

3. **Strafe / speed detection:** `strafe` is true if strafe key, mouse strafe button, or joystick strafe button is held. `speed` is true if the run key or joystick speed button is held.

4. **Turn acceleration:** If any turn input has been held for fewer than `SLOWTURNTICS` (6) tics, `tspeed = 2` (slow turn index, `angleturn[2] = 320`). After that, `tspeed = speed` (0 = normal `640`, 1 = fast `1280`).

5. **Turn / strafe from keyboard and joystick:**
   - In strafe mode: right/left key and positive/negative joystick X add to `side`.
   - In normal mode: right/left key and joystick X modify `cmd->angleturn`.

6. **Forward/back:** Up/down keys and joystick Y add to `forward`.

7. **Dedicated strafe keys:** `key_straferight`/`key_strafeleft` always modify `side` regardless of strafe-modifier state.

8. **Chat character:** Dequeues one character from `HU_dequeueChatChar()` into `cmd->chatchar`.

9. **Attack / use buttons:** Fire key/button sets `BT_ATTACK`. Use key/joystick button sets `BT_USE` and resets the double-click counter.

10. **Weapon selection:** Scans keys `'1'` through `'1'+NUMWEAPONS-2`; the first depressed key sets `BT_CHANGE` and the weapon index in bits `BT_WEAPONSHIFT`.

11. **Mouse forward:** `mousebforward` adds `forwardmove[speed]` to `forward`.

12. **Double-click for use:** Two clicks of `mousebforward` within 20 tics sets `BT_USE`. Same logic for strafe double-click (`mousebstrafe` or `joybstrafe`).

13. **Mouse look:** `mousey` is added to `forward`; in strafe mode `mousex*2` goes to `side`, otherwise `mousex*0x8` (scaled) modifies `cmd->angleturn` in the negative direction. Both `mousex` and `mousey` are zeroed afterward.

14. **Clamping:** `forward` and `side` are clamped to ±`MAXPLMOVE`.

15. **Special commands:** `sendpause` injects `BT_SPECIAL | BTS_PAUSE` (overwriting the entire buttons field). `sendsave` injects `BT_SPECIAL | BTS_SAVEGAME | (savegameslot << BTS_SAVESHIFT)`.

**Returns:** void

---

### `G_DoLoadLevel`

```c
void G_DoLoadLevel(void)
```

**Purpose:** Performs the actual level load. Sets the sky texture, resets player states, calls `P_SetupLevel`, and clears all input state.

**Key logic:**
1. Determines `skyflatnum` via `R_FlatNumForName(SKYFLATNAME)`.
2. Selects `skytexture` based on `gamemode` and `gamemap` (SKY1/SKY2/SKY3 thresholds at maps 12 and 21 for DOOM II).
3. Records `levelstarttic = gametic`.
4. Forces a screen wipe if `wipegamestate == GS_LEVEL`.
5. Transitions `gamestate = GS_LEVEL`.
6. Resets dead players to `PST_REBORN` and zeros their frag counts.
7. Calls `P_SetupLevel(gameepisode, gamemap, 0, gameskill)`.
8. Sets `displayplayer = consoleplayer`, captures `starttime`, clears `gameaction`.
9. Zeroes all input state: `gamekeydown`, `joyxmove`/`joyymove`, `mousex`/`mousey`, `sendpause`/`sendsave`/`paused`, `mousebuttons`, `joybuttons`.

**Returns:** void

---

### `G_Responder`

```c
boolean G_Responder(event_t* ev)
```

**Purpose:** Master event handler; routes input to the appropriate subsystem or records raw state.

**Key logic:**

1. **Spy mode (F12):** In `GS_LEVEL`, cycles `displayplayer` through active players.

2. **Demo/title-screen popup:** If no `gameaction` pending and in demo playback or demo screen, any key/mouse/joystick press starts the menu via `M_StartControlPanel`.

3. **In-level routing:** Tries `HU_Responder`, `ST_Responder`, `AM_Responder` in order; first that returns true consumes the event.

4. **Finale routing:** Tries `F_Responder`.

5. **Raw input recording:**
   - `ev_keydown`: sets `sendpause` for `KEY_PAUSE`; sets `gamekeydown[ev->data1] = true`; always returns true (eats event).
   - `ev_keyup`: clears `gamekeydown[ev->data1]`; returns false (lets event propagate for non-game uses).
   - `ev_mouse`: unpacks buttons (bits 0–2) into `mousebuttons[0..2]`; scales `data2`/`data3` by `(mouseSensitivity+5)/10` into `mousex`/`mousey`; returns true.
   - `ev_joystick`: unpacks buttons (bits 0–3) into `joybuttons[0..3]`; stores raw axis values; returns true.

**Returns:** `true` if the event was consumed.

---

### `G_Ticker`

```c
void G_Ticker(void)
```

**Purpose:** The master per-tic update function, the heartbeat of the game. Called once per tic from `D_DoomLoop`.

**Key logic:**

**Phase 1 — Player reborns:**
For every active player in `PST_REBORN` state, calls `G_DoReborn(i)`.

**Phase 2 — `gameaction` dispatch (while loop):**
Executes deferred actions until `gameaction == ga_nothing`:

| `gameaction` | Action taken |
|-------------|--------------|
| `ga_loadlevel` | `G_DoLoadLevel()` |
| `ga_newgame` | `G_DoNewGame()` |
| `ga_loadgame` | `G_DoLoadGame()` |
| `ga_savegame` | `G_DoSaveGame()` |
| `ga_playdemo` | `G_DoPlayDemo()` |
| `ga_completed` | `G_DoCompleted()` |
| `ga_victory` | `F_StartFinale()` |
| `ga_worlddone` | `G_DoWorldDone()` |
| `ga_screenshot` | `M_ScreenShot(); gameaction = ga_nothing` |

**Phase 3 — Ticcmd distribution:**
Computes `buf = (gametic / ticdup) % BACKUPTICS`. For each active player:
- Copies `netcmds[i][buf]` into `players[i].cmd`.
- If `demoplayback`: calls `G_ReadDemoTiccmd(cmd)` to override with recorded data.
- If `demorecording`: calls `G_WriteDemoTiccmd(cmd)` to record and verify.
- Turbo cheat check: if `cmd->forwardmove > TURBOTHRESHOLD` and the tic modulo conditions are met (every 32 tics, cycling through players), broadcasts `"{player} is turbo!"` as a HUD message on `consoleplayer`.

**Phase 4 — Network consistency:**
For net games (not netdemo), every `ticdup` tics: verifies `consistancy[i][buf] == cmd->consistancy` (calls `I_Error` on mismatch). Then updates consistency: stores `players[i].mo->x` if the player object exists, otherwise `rndindex`.

**Phase 5 — Special button handling:**
For each active player: if `cmd->buttons & BT_SPECIAL`:
- `BTS_PAUSE`: toggles `paused`, calls `S_PauseSound()` / `S_ResumeSound()`.
- `BTS_SAVEGAME`: extracts slot from `(buttons & BTS_SAVEMASK) >> BTS_SAVESHIFT`, sets `savedescription = "NET GAME"` if empty, sets `gameaction = ga_savegame`.

**Phase 6 — Per-gamestate ticking:**

| `gamestate` | Functions called |
|-------------|-----------------|
| `GS_LEVEL` | `P_Ticker()`, `ST_Ticker()`, `AM_Ticker()`, `HU_Ticker()` |
| `GS_INTERMISSION` | `WI_Ticker()` |
| `GS_FINALE` | `F_Ticker()` |
| `GS_DEMOSCREEN` | `D_PageTicker()` |

**Returns:** void

---

### `G_InitPlayer`

```c
void G_InitPlayer(int player)
```

**Purpose:** Initializes a player slot at game start. Currently just calls `G_PlayerReborn`.

**Parameters:** `player` — player index (0–3)

**Returns:** void

---

### `G_PlayerFinishLevel`

```c
void G_PlayerFinishLevel(int player)
```

**Purpose:** Clears transient player state at level completion. Zeros `powers[]` and `cards[]`, cancels invisibility (`MF_SHADOW`), resets `extralight`, `fixedcolormap`, `damagecount`, and `bonuscount`.

**Parameters:** `player` — player index

**Returns:** void

---

### `G_PlayerReborn`

```c
void G_PlayerReborn(int player)
```

**Purpose:** Resets a player to their starting inventory after death (or at a new game). Preserves frags, kill count, item count, and secret count across the reset. Zeros the entire `player_t` struct, then restores preserved values and sets default inventory: 100 HP, pistol + fist, 50 bullets.

**Key logic:**
1. Saves `frags[]`, `killcount`, `itemcount`, `secretcount`.
2. `memset` the whole `player_t` to zero.
3. Restores saved values.
4. Sets `usedown = attackdown = true` (prevents immediate fire on spawn).
5. Sets `playerstate = PST_LIVE`.
6. Sets `health = MAXHEALTH` (100).
7. Sets `readyweapon = pendingweapon = wp_pistol`.
8. Marks `wp_fist` and `wp_pistol` as owned.
9. Sets `ammo[am_clip] = 50`.
10. Sets all `maxammo[]` from the global `maxammo[]` table.

**Returns:** void

---

### `G_CheckSpot`

```c
boolean G_CheckSpot(int playernum, mapthing_t* mthing)
```

**Purpose:** Tests whether a player can safely respawn at the given map thing position.

**Parameters:**
- `playernum` — player to respawn
- `mthing` — candidate spawn spot

**Returns:** `true` if the spot is valid.

**Key logic:**
1. If the player has no `mo` yet (first spawn), checks that no other already-spawned player occupies the same grid position; returns `true` if clear.
2. Calls `P_CheckPosition` to verify no solid geometry or monster blocks the spot.
3. Manages the body queue: if `bodyqueslot >= BODYQUESIZE`, removes the oldest corpse via `P_RemoveMobj`; enqueues the current player object.
4. Spawns a teleport fog (`MT_TFOG`) offset 20 units ahead of the spawn angle.
5. Plays `sfx_telept` if this is not the very first frame (`players[consoleplayer].viewz != 1`).

---

### `G_DeathMatchSpawnPlayer`

```c
void G_DeathMatchSpawnPlayer(int playernum)
```

**Purpose:** Spawns a player at a random valid deathmatch start. Tries up to 20 random spots; falls back to the regular player start if none pass `G_CheckSpot`.

**Returns:** void

---

### `G_DoReborn`

```c
void G_DoReborn(int playernum)
```

**Purpose:** Handles the transition from `PST_REBORN` to a live player.

**Key logic:**
- **Single player:** Sets `gameaction = ga_loadlevel` (restarts the level from scratch).
- **Net game, deathmatch:** Calls `G_DeathMatchSpawnPlayer`.
- **Net game, cooperative:** Tries the player's own start; if occupied, cycles through all other player starts (temporarily faking the start type); falls back to own start unconditionally.

**Returns:** void

---

### `G_ScreenShot`

```c
void G_ScreenShot(void)
```

**Purpose:** Sets `gameaction = ga_screenshot` to schedule a screenshot at the next tic.

**Returns:** void

---

### `G_ExitLevel`

```c
void G_ExitLevel(void)
```

**Purpose:** Triggers a normal exit: `secretexit = false; gameaction = ga_completed`.

---

### `G_SecretExitLevel`

```c
void G_SecretExitLevel(void)
```

**Purpose:** Triggers a secret exit. In German-edition DOOM II without Wolf3D maps (lump `map31` absent), silently falls back to a normal exit.

---

### `G_DoCompleted`

```c
void G_DoCompleted(void)
```

**Purpose:** Executes level completion — transitions to the intermission screen.

**Key logic:**
1. Clears `gameaction`.
2. Calls `G_PlayerFinishLevel` for all active players.
3. Stops the automap if active.
4. **DOOM 1 episode-end detection:** Map 8 → `ga_victory` (end of episode); map 9 → marks all players as having found the secret.
5. **`wminfo` population:**
   - `didsecret`, `epsd` (episode−1), `last` (gamemap−1).
   - `next`: for commercial mode, handles secret exits from maps 15 and 31; normal flow is `gamemap` (0-based). For other modes, secret exit → map 9 (secret level); return from map 9 is episode-specific; normal → `gamemap`.
   - `maxkills`, `maxitems`, `maxsecret`, `maxfrags`.
   - `partime`: `35 * cpars[gamemap-1]` for DOOM II, `35 * pars[gameepisode][gamemap]` for DOOM 1.
   - `pnum = consoleplayer`.
   - Per-player stats: `skills` (kills), `sitems`, `ssecret`, `stime` (leveltime), `frags[]`.
6. Sets `gamestate = GS_INTERMISSION`, disables view and automap.
7. If `statcopy != NULL`, copies `wminfo` there.
8. Calls `WI_Start(&wminfo)`.

---

### `G_WorldDone`

```c
void G_WorldDone(void)
```

**Purpose:** Called by the intermission module when the player is done. Sets `gameaction = ga_worlddone`. In DOOM II, starts the finale for boss maps (6, 11, 20, 30) and secret exits from maps 15/31.

---

### `G_DoWorldDone`

```c
void G_DoWorldDone(void)
```

**Purpose:** Loads the next level after the intermission. Sets `gamemap = wminfo.next + 1`, calls `G_DoLoadLevel`, clears `gameaction`, sets `viewactive = true`.

---

### `G_LoadGame`

```c
void G_LoadGame(char* name)
```

**Purpose:** Schedules a save load: copies `name` into `savename`, sets `gameaction = ga_loadgame`.

---

### `G_DoLoadGame`

```c
void G_DoLoadGame(void)
```

**Purpose:** Executes the deferred save load.

**Save file layout:**

```
Offset 0                     : SAVESTRINGSIZE (24) bytes — user description string
Offset 24                    : VERSIONSIZE (16) bytes — "version NNN" string
Offset 40                    : 1 byte — gameskill
Offset 41                    : 1 byte — gameepisode
Offset 42                    : 1 byte — gamemap
Offset 43                    : MAXPLAYERS (4) bytes — playeringame[] flags
Offset 47                    : 3 bytes — leveltime (big-endian: A<<16 | B<<8 | C)
Offset 50+                   : P_UnArchivePlayers data
                               P_UnArchiveWorld data
                               P_UnArchiveThinkers data
                               P_UnArchiveSpecials data
Last byte                    : 0x1d — consistency sentinel
```

**Key logic:**
1. Reads file via `M_ReadFile`.
2. Validates version string; returns silently on mismatch.
3. Reads skill/episode/map/playeringame.
4. Calls `G_InitNew` to load the base level.
5. Reads `leveltime` from 3 bytes.
6. Calls the four `P_UnArchive*` functions to restore game objects.
7. Verifies sentinel byte `0x1d`; calls `I_Error` if wrong.
8. Frees buffer, runs `R_ExecuteSetViewSize` if size changed, calls `R_FillBackScreen`.

---

### `G_SaveGame`

```c
void G_SaveGame(int slot, char* description)
```

**Purpose:** Initiates a save: stores slot and description, sets `sendsave = true`. The save itself happens in `G_DoSaveGame` at the next tic, after the request has propagated through the ticcmd system (ensuring all net players save simultaneously).

---

### `G_DoSaveGame`

```c
void G_DoSaveGame(void)
```

**Purpose:** Writes the current game state to disk.

**Key logic:**
1. Forms filename: `SAVEGAMENAME{slot}.dsg` (or `c:\doomdata\...` with `-cdrom`).
2. Uses `screens[1] + 0x4000` as the in-memory buffer (borrows video memory not currently displayed).
3. Writes description (24 bytes), version string (16 bytes), gameskill, gameepisode, gamemap, playeringame[] flags, leveltime (3 bytes big-endian).
4. Calls `P_ArchivePlayers`, `P_ArchiveWorld`, `P_ArchiveThinkers`, `P_ArchiveSpecials`.
5. Writes sentinel `0x1d`.
6. Checks `length <= SAVEGAMESIZE`; calls `I_Error` on overflow.
7. Calls `M_WriteFile` to flush to disk.
8. Clears `gameaction`, clears `savedescription`, sets player HUD message to `GGSAVED`.
9. Calls `R_FillBackScreen`.

---

### `G_DeferedInitNew`

```c
void G_DeferedInitNew(skill_t skill, int episode, int map)
```

**Purpose:** Stores new-game parameters in `d_skill`/`d_episode`/`d_map` and sets `gameaction = ga_newgame`.

---

### `G_DoNewGame`

```c
void G_DoNewGame(void)
```

**Purpose:** Executes a deferred new game. Clears demo/net/deathmatch flags, resets multiplayer slots 1–3, clears `-respawn`/`-fast`/`-nomonsters` flags (these come from the original command-line parse; a new game from the menu resets them), forces `consoleplayer = 0`, then calls `G_InitNew`.

---

### `G_InitNew`

```c
void G_InitNew(skill_t skill, int episode, int map)
```

**Purpose:** The core new-game initializer. Clamps all parameters, adjusts monster speed for nightmare/fast mode, initializes players to `PST_REBORN`, sets all game flags, selects the sky texture, and calls `G_DoLoadLevel`.

**Key logic:**

1. **Resume if paused:** Calls `S_ResumeSound()`.
2. **Parameter clamping:**
   - Skill clamped to `sk_nightmare` maximum.
   - Episode minimum 1; maximum depends on `gamemode`: retail → 4, shareware → 1, other → 3.
   - Map: minimum 1; maximum 9 for non-commercial modes.
3. **Random seed:** Calls `M_ClearRandom()`.
4. **Monster behavior:**
   - If `skill == sk_nightmare || respawnparm`: sets `respawnmonsters = true`.
   - If `fastparm || (entering nightmare from non-nightmare)`: halves all Demon (`S_SARG_*`) state tics, sets Bruiser/Cacodemon/Imp shot speeds to `20*FRACUNIT`.
   - If `leaving nightmare to normal`: doubles Demon state tics, restores shot speeds to 15/10/10.
5. **Player init:** All players → `PST_REBORN`.
6. **Flags:** `usergame = true`, `demoplayback = false`, `paused = false`, etc.
7. **Sky texture:** Commercial → SKY1/SKY2/SKY3 by map threshold; non-commercial → SKY1–SKY4 by episode.
8. **Level load:** Calls `G_DoLoadLevel()`.

---

### `G_ReadDemoTiccmd`

```c
void G_ReadDemoTiccmd(ticcmd_t* cmd)
```

**Purpose:** Reads the next tic command from the demo stream into `cmd`.

**Demo stream format (4 bytes per tic):**

| Byte | Field | Notes |
|------|-------|-------|
| 0 | `forwardmove` | Signed byte |
| 1 | `sidemove` | Signed byte |
| 2 | `angleturn` | Unsigned byte, shifted left 8 bits on read |
| 3 | `buttons` | Unsigned byte |

If the first byte equals `DEMOMARKER` (0x80), calls `G_CheckDemoStatus()` and returns without reading.

---

### `G_WriteDemoTiccmd`

```c
void G_WriteDemoTiccmd(ticcmd_t* cmd)
```

**Purpose:** Writes the current tic command to the demo buffer, then verifies it by reading it back (ensuring the stored data exactly matches what will be played back).

**Key logic:**
1. If `'q'` key is down, calls `G_CheckDemoStatus()` to end recording.
2. Writes 4 bytes: `forwardmove`, `sidemove`, `(angleturn + 128) >> 8`, `buttons`.
3. Rewinds `demo_p` by 4 to point at the just-written data.
4. Checks for buffer overflow (`demo_p > demoend - 16`); if out of space, calls `G_CheckDemoStatus()` and returns.
5. Calls `G_ReadDemoTiccmd(cmd)` on the same 4 bytes, advancing `demo_p` — this ensures `cmd` matches what will be replayed and advances the pointer past the written tic.

**Note:** The angle encoding: write stores `(angleturn + 128) >> 8`; read reconstructs as `(unsigned char) << 8`. The +128 rounds to the nearest byte, and the byte shift preserves the top 8 bits of the 16-bit angle.

---

### `G_RecordDemo`

```c
void G_RecordDemo(char* name)
```

**Purpose:** Allocates the demo recording buffer and sets up state for recording.

**Key logic:**
1. Sets `usergame = false`.
2. Copies `name` into `demoname`, appends `.lmp`.
3. Default buffer size 128 KB (`0x20000`); overridden by `-maxdemo N` (N kilobytes).
4. Allocates buffer from zone heap at `PU_STATIC`.
5. Sets `demorecording = true`.

---

### `G_BeginRecording`

```c
void G_BeginRecording(void)
```

**Purpose:** Writes the demo header into `demobuffer` and sets `demo_p` to point past it.

**Header layout (9 + MAXPLAYERS bytes):**

| Offset | Content |
|--------|---------|
| 0 | `VERSION` |
| 1 | `gameskill` |
| 2 | `gameepisode` |
| 3 | `gamemap` |
| 4 | `deathmatch` |
| 5 | `respawnparm` |
| 6 | `fastparm` |
| 7 | `nomonsters` |
| 8 | `consoleplayer` |
| 9–12 | `playeringame[0..3]` |

---

### `G_DeferedPlayDemo`

```c
void G_DeferedPlayDemo(char* name)
```

**Purpose:** Stores the demo lump name in `defdemoname` and sets `gameaction = ga_playdemo`.

---

### `G_DoPlayDemo`

```c
void G_DoPlayDemo(void)
```

**Purpose:** Reads the demo header and initializes game state for playback.

**Key logic:**
1. Sets `demobuffer = demo_p = W_CacheLumpName(defdemoname, PU_STATIC)`.
2. Reads and validates `VERSION`; prints a warning and returns on mismatch.
3. Reads header fields: skill, episode, map, deathmatch, respawnparm, fastparm, nomonsters, consoleplayer, playeringame[].
4. If `playeringame[1]` is set, marks as `netgame = netdemo = true`.
5. Sets `precache = false` (skips texture precaching for speed), calls `G_InitNew`, restores `precache = true`.
6. Sets `usergame = false`, `demoplayback = true`.

---

### `G_TimeDemo`

```c
void G_TimeDemo(char* name)
```

**Purpose:** Starts a benchmark timing run. Sets `nodrawers` and `noblit` from command-line flags, sets `timingdemo = singletics = true`, then defers demo playback.

---

### `G_CheckDemoStatus`

```c
boolean G_CheckDemoStatus(void)
```

**Purpose:** Handles end-of-demo cleanup for all three modes (timing, playback, recording).

**Key logic:**

**Timing mode (`timingdemo == true`):**
- Computes elapsed real tics (`I_GetTime() - starttime`).
- Calls `I_Error("timed %i gametics in %i realtics", ...)` to terminate and report.

**Playback mode (`demoplayback == true`):**
- If `singledemo`: calls `I_Quit()`.
- Changes demo buffer zone tag to `PU_CACHE` (allows it to be freed).
- Resets: `demoplayback`, `netdemo`, `netgame`, `deathmatch`, `playeringame[1..3]`, `respawnparm`, `fastparm`, `nomonsters`, `consoleplayer`.
- Calls `D_AdvanceDemo()` to move to the next demo in the attract loop.
- Returns `true`.

**Recording mode (`demorecording == true`):**
- Writes `DEMOMARKER` (0x80) at `demo_p`.
- Calls `M_WriteFile(demoname, demobuffer, demo_p - demobuffer)`.
- Frees `demobuffer` via `Z_Free`.
- Sets `demorecording = false`.
- Calls `I_Error("Demo %s recorded", demoname)` to terminate and report.

**Returns:** `true` if a demo loop action was taken (caller should return); `false` otherwise.

---

## Data Structures

### Par time tables

```c
int pars[4][10] = {
    {0},
    {0, 30, 75, 120, 90, 165, 180, 180, 30, 165},  // Episode 1
    {0, 90, 90,  90,120,  90, 360, 240, 30, 170},  // Episode 2
    {0, 90, 45,  90,150,  90,  90, 165, 30, 135}   // Episode 3
};

int cpars[32] = {
    30, 90,120,120, 90,150,120,120,270, 90,   //  1-10
   210,150,150,150,210,150,420,150,210,150,   // 11-20
   240,150,180,150,150,300,330,420,300,180,   // 21-30
   120, 30                                    // 31-32
};
```

All values in seconds; multiplied by 35 (tics/second) to get tic-based par times.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `doomdef.h` | Basic types, constants, `skill_t`, `MAXPLAYERS`, `FRACUNIT`, etc. |
| `doomstat.h` | `gamemode`, `automapactive`, `mouseSensitivity`, `leveltime`, `netcmds[]`, `ticdup`, `maketic`, `rndindex`, `deathmatchstarts`, `deathmatch_p`, `playerstarts`, `maxammo[]`, `respawnparm`, `fastparm`, `nomonsters`, `singletics` |
| `z_zone.h` | `Z_Malloc`, `Z_Free`, `Z_ChangeTag`, `Z_CheckHeap` for demo buffer and zone checks |
| `f_finale.h` | `F_StartFinale`, `F_Ticker`, `F_Responder` |
| `m_argv.h` | `M_CheckParm`, `myargc`, `myargv` for command-line options |
| `m_misc.h` | `M_ReadFile`, `M_WriteFile`, `M_ScreenShot`, `M_ClearRandom` |
| `m_menu.h` | `M_StartControlPanel` (opened on demo-screen keypresses) |
| `m_random.h` | `P_Random` (deathmatch spot selection) |
| `i_system.h` | `I_Error`, `I_Quit`, `I_GetTime`, `I_BaseTiccmd` |
| `p_setup.h` | `P_SetupLevel` |
| `p_saveg.h` | `P_ArchivePlayers/World/Thinkers/Specials`, `P_UnArchive*`, `save_p` |
| `p_tick.h` | `P_Ticker` |
| `p_local.h` | `P_CheckPosition`, `P_SpawnMobj`, `P_RemoveMobj`, `finecosine[]`, `finesine[]` |
| `d_main.h` | `D_AdvanceDemo`, `D_PageTicker`, `wipegamestate`, `pagename` |
| `wi_stuff.h` | `WI_Start`, `WI_Ticker` |
| `hu_stuff.h` | `HU_Responder`, `HU_Ticker`, `HU_dequeueChatChar` |
| `st_stuff.h` | `ST_Responder`, `ST_Ticker` |
| `am_map.h` | `AM_Responder`, `AM_Ticker`, `AM_Stop` |
| `v_video.h` | `screens[]` (used as save buffer via `screens[1]`) |
| `w_wad.h` | `W_CacheLumpName`, `W_CheckNumForName` |
| `s_sound.h` | `S_PauseSound`, `S_ResumeSound`, `S_StartSound` |
| `dstrings.h` | `GGSAVED` (save confirmation message), `SAVEGAMENAME` |
| `sounds.h` | `sfx_telept` |
| `r_data.h` | `R_FlatNumForName`, `R_TextureNumForName`, `R_FillBackScreen` |
| `r_sky.h` | `skyflatnum`, `skytexture`, `SKYFLATNAME` |
| `r_main.h` (extern) | `setsizeneeded`, `R_ExecuteSetViewSize` |
| `p_inter.h` (via doomstat) | `player_names[]` (turbo message formatting) |
| `info.h` (via p_local/p_mobj) | `states[]`, `mobjinfo[]`, `MT_TFOG` |
