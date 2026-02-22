# File Overview

`st_stuff.c` is the main status bar implementation for the DOOM engine. It owns the full 320x32 pixel HUD bar at the bottom of the screen and orchestrates everything it displays: the player's health, armor, ammo, weapon inventory, key cards, the animated face/status indicator, the frags count in deathmatch, and palette-shift effects for damage and item pickups. It also processes all keyboard cheat codes.

The status bar is implemented using the widget library from `st_lib.c`. On initialization it creates a set of `st_number_t`, `st_percent_t`, `st_multicon_t`, and `st_binicon_t` widgets, each pointing directly into the player's data structures. Each tick, `ST_Ticker` updates widget state; each frame, `ST_Drawer` decides whether to do a full refresh or an incremental update.

A secondary screen buffer (screen 4, `BG`) is used as a static copy of the status bar background graphic. When a widget needs to erase its previous value, it copies from `BG` back to `FG` (screen 0). This avoids reloading the background patch every frame.

---

## Global Variables

All variables in this file are `static` (file-scope only) unless noted.

### Cheat Code Data (Global — accessible externally)

These arrays contain the encoded keystroke sequences for each cheat code. The encoding obscures the sequences from trivial inspection (each byte is an encoded key value):

| Type | Name | Cheat |
|------|------|-------|
| `unsigned char[]` | `cheat_mus_seq[]` | `IDMUS` — change music |
| `unsigned char[]` | `cheat_choppers_seq[]` | `IDCHOPPERS` — chainsaw + invulnerability |
| `unsigned char[]` | `cheat_god_seq[]` | `IDDQD` — god mode |
| `unsigned char[]` | `cheat_ammo_seq[]` | `IDKFA` — full ammo + keys + armor |
| `unsigned char[]` | `cheat_ammonokey_seq[]` | `IDFA` — full ammo + armor (no keys) |
| `unsigned char[]` | `cheat_noclip_seq[]` | `IDSPISPOPD` — no-clip (Doom 1) |
| `unsigned char[]` | `cheat_commercial_noclip_seq[]` | `IDCLIP` — no-clip (Doom 2) |
| `unsigned char[][10]` | `cheat_powerup_seq[7]` | `BEHOLD?` — power-up cheats (6 specific + menu) |
| `unsigned char[]` | `cheat_clev_seq[]` | `IDCLEV` — change level |
| `unsigned char[]` | `cheat_mypos_seq[]` | `IDMYPOS` — show position |

| Type | Name | Notes |
|------|------|-------|
| `cheatseq_t` | `cheat_mus` | Cheat state for music cheat |
| `cheatseq_t` | `cheat_god` | Cheat state for god mode |
| `cheatseq_t` | `cheat_ammo` | Cheat state for IDKFA |
| `cheatseq_t` | `cheat_ammonokey` | Cheat state for IDFA |
| `cheatseq_t` | `cheat_noclip` | Cheat state for IDSPISPOPD |
| `cheatseq_t` | `cheat_commercial_noclip` | Cheat state for IDCLIP |
| `cheatseq_t[7]` | `cheat_powerup` | Cheat states for 7 BEHOLD variants |
| `cheatseq_t` | `cheat_choppers` | Cheat state for IDCHOPPERS |
| `cheatseq_t` | `cheat_clev` | Cheat state for IDCLEV |
| `cheatseq_t` | `cheat_mypos` | Cheat state for IDMYPOS |

### Static Player and State Variables

| Type | Name | Description |
|------|------|-------------|
| `player_t*` | `plyr` | Pointer to the console player's data structure |
| `boolean` | `st_firsttime` | True immediately after `ST_Start()`; forces full refresh |
| `int` | `veryfirsttime` | 1 until `ST_Init()` runs; prevents double initialization |
| `int` | `lu_palette` | WAD lump number for `PLAYPAL` (palette data) |
| `unsigned int` | `st_clock` | Tick counter incremented each `ST_Ticker` call |
| `int` | `st_msgcounter` | Countdown for temporary message display |
| `st_chatstateenum_t` | `st_chatstate` | Current chat input state (`StartChatState`, `WaitDestState`, `GetChatState`) |
| `st_stateenum_t` | `st_gamestate` | Whether in `AutomapState` or `FirstPersonState` |
| `boolean` | `st_statusbaron` | Whether the status bar is shown (false in full-screen with no automap) |
| `boolean` | `st_chat` | Whether chat mode is active |
| `boolean` | `st_oldchat` | Saved chat state before message overlay |
| `boolean` | `st_cursoron` | Whether the chat cursor is visible |
| `boolean` | `st_notdeathmatch` | `!deathmatch` — controls visibility of weapons panel |
| `boolean` | `st_armson` | `!deathmatch && st_statusbaron` — weapons panel visibility |
| `boolean` | `st_fragson` | `deathmatch && st_statusbaron` — frags counter visibility |
| `int` | `st_palette` | Currently applied palette index (for avoiding redundant palette switches) |

### Static Graphic Patch Arrays

| Type | Name | Description |
|------|------|-------------|
| `patch_t*` | `sbar` | Main status bar background (`STBAR`) |
| `patch_t*[10]` | `tallnum` | Large digit patches 0-9 (`STTNUM0`-`STTNUM9`) |
| `patch_t*` | `tallpercent` | Large percent sign (`STTPRCNT`) |
| `patch_t*[10]` | `shortnum` | Small yellow digit patches 0-9 (`STYSNUM0`-`STYSNUM9`) |
| `patch_t*[NUMCARDS]` | `keys` | Key card icon patches (`STKEYS0`-`STKEYS5`) |
| `patch_t*[ST_NUMFACES]` | `faces` | All face state patches (42 total: 5 pain levels x 8 states + 2 extra) |
| `patch_t*` | `faceback` | Face background for multiplayer (`STFB0`-`STFB3`) |
| `patch_t*` | `armsbg` | Weapons panel background (`STARMS`) |
| `patch_t*[6][2]` | `arms` | Weapon indicator patches: `[weapon][0]`=gray (unowned), `[weapon][1]`=yellow (owned) |

### Static Widget Variables

| Type | Name | Description |
|------|------|-------------|
| `st_number_t` | `w_ready` | Ready weapon ammo display |
| `st_number_t` | `w_frags` | Deathmatch frags count |
| `st_percent_t` | `w_health` | Health percentage display |
| `st_binicon_t` | `w_armsbg` | Weapons panel background icon |
| `st_multicon_t[6]` | `w_arms` | 6 weapon ownership indicators |
| `st_multicon_t` | `w_faces` | Face/status indicator |
| `st_multicon_t[3]` | `w_keyboxes` | 3 key slot indicators |
| `st_percent_t` | `w_armor` | Armor percentage display |
| `st_number_t[4]` | `w_ammo` | Current ammo counts (4 ammo types) |
| `st_number_t[4]` | `w_maxammo` | Maximum ammo counts (4 ammo types) |

### Static Face Logic Variables

| Type | Name | Description |
|------|------|-------------|
| `int` | `st_fragscount` | Computed frag sum for the current player |
| `int` | `st_oldhealth` | Health value from the previous tick (for pain detection) |
| `boolean[NUMWEAPONS]` | `oldweaponsowned` | Weapon ownership from previous tick (for evil grin detection) |
| `int` | `st_facecount` | Tics remaining before the current face expression can change |
| `int` | `st_faceindex` | Index into `faces[]` for the currently displayed face |
| `int[3]` | `keyboxes` | Current key slot display values (card type index or -1 for empty) |
| `int` | `st_randomnumber` | A fresh random number each tick (from `M_Random`) |
| `boolean` | `st_stopped` | Whether the status bar is currently stopped (between levels) |

---

## Layout Constants (Defines)

All positions are in pixels on the 320x200 screen (status bar occupies rows 168-199):

| Name | Value | Description |
|------|-------|-------------|
| `STARTREDPALS` | 1 | First palette index for red damage tint |
| `STARTBONUSPALS` | 9 | First palette index for gold item pickup tint |
| `NUMREDPALS` | 8 | Number of red pain palettes |
| `NUMBONUSPALS` | 4 | Number of bonus palettes |
| `RADIATIONPAL` | 13 | Palette index for radiation suit green tint |
| `ST_FACEPROBABILITY` | 96 | Out of 256; chance per tick that idle face will change direction |
| `ST_NUMPAINFACES` | 5 | Five pain levels (0=healthy, 4=near death) |
| `ST_NUMSTRAIGHTFACES` | 3 | Three forward-facing variations per pain level |
| `ST_NUMTURNFACES` | 2 | Two turned-head faces per pain level |
| `ST_NUMSPECIALFACES` | 3 | Ouch + evil grin + rampage per pain level |
| `ST_NUMFACES` | 42 | Total face patches (5*8 + god + dead) |
| `ST_EVILGRINCOUNT` | `2*TICRATE` | Duration of evil grin on weapon pickup |
| `ST_RAMPAGEDELAY` | `2*TICRATE` | Tics of sustained fire before rampage face |
| `ST_MUCHPAIN` | 20 | Health delta threshold for "ouch" face (large hit) |

---

## Functions

### `ST_refreshBackground`

```c
void ST_refreshBackground(void)
```

**Purpose:** Draws the status bar background graphic to screen `BG` (screen 4) and copies it to screen `FG` (screen 0).

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Draws `sbar` into `BG` at `(ST_X, 0)`. In netgame, also draws the `faceback` patch. Then blits the entire status bar region from `BG` to `FG` via `V_CopyRect`.

---

### `ST_Responder`

```c
boolean ST_Responder(event_t* ev)
```

**Purpose:** Processes input events for the status bar. Handles cheat code detection and automap state transitions.

**Parameters:**
- `ev` — Pointer to the input event.

**Returns:** `false` always (the status bar never consumes events from the main loop).

**Key Logic:**
1. Detects `AM_MSGENTERED` / `AM_MSGEXITED` key-up events to switch between `AutomapState` and `FirstPersonState`.
2. For key-down events in non-netgame single-player mode, checks all cheat sequences using `cht_CheckCheat`:
   - `IDDQD`: toggles god mode (`CF_GODMODE`), sets health to 100 if enabling.
   - `IDFA`: grants 200% armor, all weapons, full ammo.
   - `IDKFA`: same as IDFA plus all key cards.
   - `IDMUS`: changes music to the specified track number.
   - `IDSPISPOPD` / `IDCLIP`: toggles noclip mode (`CF_NOCLIP`).
   - `BEHOLD?` (6 variants): toggles power-ups (invulnerability, strength, invisibility, radiation suit, automap, light amp).
   - `BEHOLD` (7th): displays the power-up menu message.
   - `IDCHOPPERS`: grants chainsaw + invulnerability.
   - `IDMYPOS`: displays current position and angle as a message.
3. `IDCLEV` (level warp) is checked in all modes including netgame. Validates the episode/map combination against the current game mode before calling `G_DeferedInitNew`.

---

### `ST_calcPainOffset`

```c
int ST_calcPainOffset(void)
```

**Purpose:** Calculates the base index offset into the `faces[]` array corresponding to the player's current health level (pain level).

**Parameters:** None.

**Returns:** Integer offset for the appropriate pain tier in the face array.

**Key Logic:** Clamps health to 100, then computes `ST_FACESTRIDE * ((100 - health) * ST_NUMPAINFACES / 101)`. This maps 0-19 health to pain level 4, 20-39 to level 3, etc. Caches the result to avoid recalculation when health hasn't changed.

---

### `ST_updateFaceWidget`

```c
void ST_updateFaceWidget(void)
```

**Purpose:** Runs the face state machine each tick, selecting which of the 42 face patches to display based on a priority hierarchy of game events.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic (priority order, highest first):**
1. **Priority 9 — Dead:** If `plyr->health == 0`, shows `ST_DEADFACE`, locks count at 1.
2. **Priority 8 — Evil Grin:** If `plyr->bonuscount` is nonzero and a new weapon was just picked up (comparing `oldweaponsowned[]`), shows the evil grin face for `ST_EVILGRINCOUNT` tics.
3. **Priority 7 — Being Attacked with Big Hit:** If taking damage from an external attacker and the health drop exceeds `ST_MUCHPAIN`, shows the "ouch" face.
4. **Priority 7 — Being Attacked (directional):** If taking damage from a known attacker, computes the relative angle to the attacker and shows a face turned toward the threat (left, right, or front/rampage).
5. **Priority 6 — Self-Damage (rampage face):** Taking damage without a tracked attacker shows the rampage face.
6. **Priority 5 — Sustained Fire (rampage):** If the player holds fire for `ST_RAMPAGEDELAY` tics, shows the rampage face.
7. **Priority 4 — God Mode / Invulnerability:** Shows the god face.
8. **Default — Idle:** When `st_facecount` reaches zero, picks a random straight-facing expression from the pain level tier, resets count to `ST_STRAIGHTFACECOUNT`.

Each state decrements `st_facecount` each tick; the face won't change until the count reaches zero (unless a higher priority event occurs).

---

### `ST_updateWidgets`

```c
void ST_updateWidgets(void)
```

**Purpose:** Updates all widget state pointers and derived values each tick.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:**
1. If the ready weapon uses no ammo, points `w_ready.num` at the sentinel value 1994 (displays nothing). Otherwise points it at `plyr->ammo[weaponinfo[readyweapon].ammo]`.
2. Updates `keyboxes[0..2]`: each slot shows the corresponding card index if the player has it, preferring skull variants (indices 3-5) over card variants (indices 0-2) for the same color.
3. Calls `ST_updateFaceWidget()`.
4. Updates boolean flags: `st_notdeathmatch`, `st_armson`, `st_fragson`.
5. Computes `st_fragscount` as the sum of frags against all other players minus own self-frags.
6. Manages the message display timer (`st_msgcounter`).

---

### `ST_Ticker`

```c
void ST_Ticker(void)
```

**Purpose:** Called each game tick to advance the status bar's internal state.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Increments `st_clock`, calls `M_Random()` for `st_randomnumber`, calls `ST_updateWidgets()`, then saves `plyr->health` to `st_oldhealth`.

---

### `ST_doPaletteStuff`

```c
void ST_doPaletteStuff(void)
```

**Purpose:** Applies palette-based screen tint effects for damage, item pickup, berserk, and radiation suit.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:**
1. Starts with `cnt = plyr->damagecount` (0-255, decrements each tick after taking damage).
2. If berserk power is active, computes a fading berserk counter `bzc = 12 - (power >> 6)` and takes the max of `bzc` and `cnt`.
3. If `cnt > 0`: selects a red palette `STARTREDPALS + clamp(cnt/8, 0, NUMREDPALS-1)`.
4. Else if `bonuscount > 0`: selects a gold/bonus palette.
5. Else if radiation suit is active (more than 4*32 tics remaining, or blinking at tics with bit 3 set): selects `RADIATIONPAL`.
6. Else: palette 0 (normal).
7. Only calls `I_SetPalette` when the palette index actually changes.

---

### `ST_drawWidgets`

```c
void ST_drawWidgets(boolean refresh)
```

**Purpose:** Calls the update function for every widget, performing a full or incremental redraw.

**Parameters:**
- `refresh` — If true, forces all widgets to redraw even if unchanged.

**Returns:** Nothing.

**Key Logic:** Calls `STlib_updateNum`, `STlib_updatePercent`, `STlib_updateBinIcon`, and `STlib_updateMultIcon` for each of the status bar's widgets.

---

### `ST_doRefresh`

```c
void ST_doRefresh(void)
```

**Purpose:** Performs a complete redraw of the status bar background and all widgets.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Clears `st_firsttime`, calls `ST_refreshBackground()` then `ST_drawWidgets(true)`.

---

### `ST_diffDraw`

```c
void ST_diffDraw(void)
```

**Purpose:** Performs an incremental (delta) redraw of only the widgets that have changed.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Calls `ST_drawWidgets(false)`.

---

### `ST_Drawer`

```c
void ST_Drawer(boolean fullscreen, boolean refresh)
```

**Purpose:** Main entry point for drawing the status bar each frame.

**Parameters:**
- `fullscreen` — If true (full-screen view), status bar is hidden unless automap is active.
- `refresh` — If true, force a complete redraw.

**Returns:** Nothing.

**Key Logic:** Sets `st_statusbaron = (!fullscreen || automapactive)`. Applies any forced refresh. Calls `ST_doPaletteStuff()`, then either `ST_doRefresh()` or `ST_diffDraw()`.

---

### `ST_loadGraphics`

```c
void ST_loadGraphics(void)
```

**Purpose:** Loads all graphic patches used by the status bar from the WAD.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Loads: tall numbers (STTNUM0-9), small numbers (STYSNUM0-9), percent sign (STTPRCNT), key card patches (STKEYS0-5), arms background (STARMS), gray/yellow weapon number patches, face background (STFB`n`), status bar background (STBAR), and all 42 face patches in a nested loop over pain levels and face states.

---

### `ST_loadData`

```c
void ST_loadData(void)
```

**Purpose:** Loads the PLAYPAL lump number and calls `ST_loadGraphics`.

---

### `ST_unloadGraphics`

```c
void ST_unloadGraphics(void)
```

**Purpose:** Demotes all loaded graphic patches to `PU_CACHE` so zone memory can reclaim them between levels.

---

### `ST_unloadData`

```c
void ST_unloadData(void)
```

**Purpose:** Wrapper that calls `ST_unloadGraphics`.

---

### `ST_initData`

```c
void ST_initData(void)
```

**Purpose:** Initializes all status bar state variables at the start of a level.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Sets `plyr` to the console player, resets all state flags, copies current weapon ownership to `oldweaponsowned[]`, clears `keyboxes[]` to -1, calls `STlib_init()`.

---

### `ST_createWidgets`

```c
void ST_createWidgets(void)
```

**Purpose:** Creates and initializes all widget instances with their screen positions, patch arrays, and data pointers.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Calls the appropriate `STlib_init*` function for each widget, binding each one to a specific screen position (using the layout `#define` constants) and a specific field in the player data structure. Widgets created:
- `w_ready` (ammo for current weapon)
- `w_health` (health percent)
- `w_armsbg` (weapons panel background)
- `w_arms[0..5]` (weapon 2-7 ownership)
- `w_frags` (deathmatch frag count)
- `w_faces` (face indicator)
- `w_armor` (armor percent)
- `w_keyboxes[0..2]` (three key slots)
- `w_ammo[0..3]` (current ammo for all 4 types)
- `w_maxammo[0..3]` (max ammo for all 4 types)

---

### `ST_Start`

```c
void ST_Start(void)
```

**Purpose:** Starts the status bar for a new level. Stops the previous session if needed.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Calls `ST_Stop()` if not already stopped, then `ST_initData()`, `ST_createWidgets()`, clears `st_stopped`.

---

### `ST_Stop`

```c
void ST_Stop(void)
```

**Purpose:** Shuts down the status bar at the end of a level. Restores the default palette.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Calls `I_SetPalette` with the base `PLAYPAL` palette (palette 0) to clear any damage/pickup tint. Sets `st_stopped = true`.

---

### `ST_Init`

```c
void ST_Init(void)
```

**Purpose:** One-time initialization at engine startup. Loads all graphics data and allocates the status bar background buffer.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Sets `veryfirsttime = 0`, calls `ST_loadData()`, allocates `screens[4]` (`ST_WIDTH * ST_HEIGHT` bytes, `PU_STATIC`) as the background screen buffer.

---

## Dependencies

| File | Purpose |
|------|---------|
| `i_system.h` | `I_Error` |
| `i_video.h` | `I_SetPalette` |
| `z_zone.h` | `Z_Malloc`, `Z_ChangeTag` |
| `m_random.h` | `M_Random()` for face randomization |
| `w_wad.h` | `W_CacheLumpName`, `W_CacheLumpNum`, `W_GetNumForName` |
| `doomdef.h` | `TICRATE`, `NUMWEAPONS`, `NUMCARDS`, `NUMAMMO`, `MAXPLAYERS`, game mode constants |
| `g_game.h` | `G_DeferedInitNew` for IDCLEV level warp |
| `st_stuff.h` | Own public header |
| `st_lib.h` | Widget library — all `STlib_*` functions and widget types |
| `r_local.h` | Rendering definitions, `R_PointToAngle2` |
| `p_local.h` | `player_t`, `mobj_t`, `players[]` |
| `p_inter.h` | `P_GivePower` for BEHOLD cheats |
| `am_map.h` | `automapactive`, `AM_MSGENTERED`, `AM_MSGEXITED`, `AM_MSGHEADER` |
| `m_cheat.h` | `cht_CheckCheat`, `cht_GetParam`, `cheatseq_t` |
| `s_sound.h` | `S_ChangeMusic` for IDMUS cheat |
| `v_video.h` | `V_DrawPatch`, `V_CopyRect`, `screens[]` |
| `doomstat.h` | `gamemode`, `gameskill`, `deathmatch`, `netgame`, `consoleplayer`, `playeringame[]` |
| `dstrings.h` | Cheat feedback message strings (`STSTR_DQDON`, etc.) |
| `sounds.h` | `musicenum_t` values for IDMUS |
