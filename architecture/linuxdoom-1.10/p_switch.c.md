# File Overview

`p_switch.c` manages wall switch and button textures. In DOOM, switches are wall textures that have a paired "pressed" appearance (e.g., `SW1BRCOM` and `SW2BRCOM`). When a player presses a switch, the texture is swapped to its alternate state. Permanent switches have their texture changed and never revert. Buttons (temporary switches) revert automatically after a fixed time (`BUTTONTIME` = 35 tics = 1 second), restored with a clicking sound.

This file also contains the main player-use dispatcher `P_UseSpecialLine`, which is called when a player presses the use key against a special linedef.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `switchlist_t[]` | `alphSwitchList` | Hard-coded table of all switch texture pairs and their episode requirements. Contains entries for DOOM shareware (episode 1), registered (episodes 2-3), and DOOM II (episode 3). Terminated by an entry with `episode == 0`. |
| `int[MAXSWITCHES*2]` | `switchlist` | Flattened array of texture number pairs, populated at startup by `P_InitSwitchList`. Pairs are stored as `[inactive_texnum, active_texnum, ...]`. |
| `int` | `numswitches` | Number of valid switch pairs in `switchlist` (not counting the -1 terminator). |
| `button_t[MAXBUTTONS]` | `buttonlist` | Array of active button presses. Each entry records the line, texture tier, original texture, and countdown timer. |

## Functions

### `P_InitSwitchList`
```c
void P_InitSwitchList(void)
```
**Purpose:** Initializes `switchlist[]` with the texture numbers for all applicable switch pairs, based on the current game mode.

**Parameters:** None.

**Key logic:**
- Sets episode level: 1 for shareware, 2 for registered DOOM, 3 for DOOM II (commercial).
- Iterates `alphSwitchList[]` until `episode == 0`.
- For each entry with `episode <= current_episode`, calls `R_TextureNumForName` for both the inactive (`name1`) and active (`name2`) texture names and stores them consecutively in `switchlist[]`.
- Sets `numswitches` to half the populated count (since pairs share the array).
- Terminates the active portion with `-1`.

### `P_StartButton`
```c
void P_StartButton(line_t* line, bwhere_e w, int texture, int time)
```
**Purpose:** Registers a new button press record so the button texture will be automatically reverted after `time` tics.

**Parameters:**
- `line` - The linedef that was activated.
- `w` - Which texture tier (`top`, `middle`, `bottom`) to revert.
- `texture` - The original (inactive) texture number to restore.
- `time` - Countdown in tics.

**Key logic:**
- First checks if this line already has an active button (prevents double-activation).
- Searches `buttonlist[]` for a slot with `btimer == 0` (inactive).
- Fills in the slot with the line, tier, original texture, timer, and a `soundorg` pointer to the front sector's sound origin.
- Calls `I_Error` if no slots are available.

### `P_ChangeSwitchTexture`
```c
void P_ChangeSwitchTexture(line_t* line, int useAgain)
```
**Purpose:** Swaps a switch wall texture to its alternate state and optionally registers a button for automatic reversion.

**Parameters:**
- `line` - The linedef containing the switch texture.
- `useAgain` - If non-zero, registers a button press (temporary switch/button). If zero, the texture change is permanent (one-time switch).

**Key logic:**
- Clears `line->special` if `useAgain == 0` (consumed single-use switch).
- Reads the top, middle, and bottom textures from `sides[line->sidenum[0]]`.
- Selects the sound: most switches play `sfx_swtchn`; exit switches (special == 11) play `sfx_swtchx`.
- Iterates all `numswitches * 2` texture pairs in `switchlist`.
- Finds which texture (top/mid/bottom) matches a switch entry.
- Changes that texture to the paired (alternate) texture via `switchlist[i^1]` (XOR with 1 to flip between pair elements).
- If `useAgain`, calls `P_StartButton` with the original texture to enable auto-revert.

### `P_UseSpecialLine`
```c
boolean P_UseSpecialLine(mobj_t* thing, line_t* line, int side)
```
**Purpose:** Called when a thing presses the use key against a linedef with a non-zero special. Dispatches the appropriate action.

**Parameters:**
- `thing` - The actor pressing the line (usually a player's mobj).
- `line` - The targeted linedef.
- `side` - Side from which the line was activated (0 = front, 1 = back).

**Return value:** `true` if the action was successfully activated; `false` otherwise.

**Key logic:**
- **Back-side check**: Only special case 124 (sliding door, unused) may be activated from the back side. All others require activation from the front side.
- **Monster restriction**: Non-player things cannot activate secret lines (`ML_SECRET` flag). Non-players can only activate manual door specials (1, 32, 33, 34).
- **Manual doors** (cases 1, 26-28, 31-34, 117-118): These immediately call `EV_VerticalDoor` and do NOT call `P_ChangeSwitchTexture` — they have no switch texture to toggle.
- **One-shot switches** (cases 7, 9, 11, 14, 15, etc.): These call an `EV_*` function and if it succeeds, call `P_ChangeSwitchTexture(line, 0)` (permanent change).
- **Buttons** (cases 42-139 with button behavior): Call `EV_*` and if it succeeds, call `P_ChangeSwitchTexture(line, 1)` (temporary, auto-revert).
- Returns `true` if the switch block was entered (even if some underlying EV calls returned 0 due to already-active sectors).

## Dependencies

| File | Reason |
|------|--------|
| `i_system.h` | `I_Error` |
| `doomdef.h` | Type definitions, `ML_SECRET`, `ML_TWOSIDED` flags |
| `p_local.h` | Door/floor/ceiling/platform EV functions, `P_AddThinker` |
| `g_game.h` | `G_ExitLevel`, `G_SecretExitLevel` |
| `s_sound.h` | `S_StartSound` |
| `sounds.h` | `sfx_swtchn`, `sfx_swtchx` |
| `doomstat.h` | `gamemode` for episode determination |
| `r_state.h` | `texturetranslation` indirectly via `R_TextureNumForName` |
