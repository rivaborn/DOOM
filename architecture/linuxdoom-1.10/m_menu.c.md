# File Overview

`m_menu.c` implements the entire in-game menu system for DOOM. This includes the main pause/escape menu and all its sub-menus, save/load game functionality, option sliders, modal message boxes, and all hotkey handling. The file is self-contained: all menu data structures, menu item arrays, and helper functions are defined internally; only the five public functions declared in `m_menu.h` are externally visible.

The menu system is driven by two internal data structures (`menuitem_t` and `menu_t`) that form a linked tree of menus. Navigation through the tree, item activation, slider adjustment, and drawing are all handled generically by a small core of functions, with per-menu behaviour delegated to draw routines and per-item callback functions.

---

## Data Structures

### `menuitem_t`
```c
typedef struct {
    short  status;           // 0=no cursor, 1=selectable, 2=slider (arrows change value), -1=spacer
    char   name[10];         // WAD lump name of the graphic patch to draw, or "" for text items
    void   (*routine)(int choice);  // callback when item is activated; choice=0 left, 1 right for sliders
    char   alphaKey;         // keyboard shortcut character for this item
} menuitem_t;
```
One entry in a menu's item list. The `status` field drives the cursor and input handling:
- `1`: Normal selectable item; `routine(itemOn)` called on Enter.
- `2`: Slider; left/right arrows call `routine(0)` and `routine(1)` respectively; Enter calls `routine(1)` (right).
- `-1`: Non-selectable spacer; cursor skips over these.
- `0`: Item exists but is currently disabled (e.g. empty save slot in load menu).

### `menu_t`
```c
typedef struct menu_s {
    short        numitems;     // number of entries in menuitems[]
    struct menu_s *prevMenu;   // parent menu (Backspace navigates here)
    menuitem_t   *menuitems;   // array of menu items
    void         (*routine)(); // draw routine for menu title/background
    short        x;            // screen X origin of item graphics
    short        y;            // screen Y origin of first item
    short        lastOn;       // which item was last active (restored on re-entry)
} menu_t;
```
Describes one entire menu page. `prevMenu` links menus into a back-navigation stack.

---

## Global Variables

### Player-configurable settings (saved to config file)

| Variable | Type | Default | Purpose |
|---|---|---|---|
| `mouseSensitivity` | `int` | read from config | Mouse sensitivity, range 0-9, adjusted by `M_ChangeSensitivity` |
| `showMessages` | `int` | read from config | 0=messages off, 1=messages on, toggled by `M_ChangeMessages` |
| `detailLevel` | `int` | read from config | 0=high detail, 1=low detail (low detail was never fully implemented) |
| `screenblocks` | `int` | read from config | Raw screen size 3-11; drives `R_SetViewSize` |

### Internal menu state

| Variable | Type | Purpose |
|---|---|---|
| `screenSize` | `int` | Derived from `screenblocks - 3`; displayed on the screen size slider (0-8) |
| `quickSaveSlot` | `int` | Which save slot to use for quicksave/quickload; `-1` = none chosen, `-2` = waiting for slot selection |
| `messageToPrint` | `int` | `1` when a message overlay is active |
| `messageString` | `char *` | Pointer to the message text to display |
| `messx`, `messy` | `int` | Screen position of the message (computed at display time) |
| `messageLastMenuActive` | `int` | Saved value of `menuactive` before the message was shown |
| `messageNeedsInput` | `boolean` | `true` if the message requires a y/n/space/Escape response |
| `messageRoutine` | `void (*)(int)` | Callback invoked with the key pressed to dismiss the message |
| `saveStringEnter` | `int` | `1` when the user is typing a save game name |
| `saveSlot` | `int` | Which save slot is being named |
| `saveCharIndex` | `int` | Cursor position in the save string being typed |
| `saveOldString[24]` | `char[]` | Backup of the save string before editing (restored on Escape) |
| `savegamestrings[10][24]` | `char[][]` | Cached descriptions for all 10 save slots |
| `inhelpscreens` | `boolean` | Set `true` while a help screen is displayed; clears status bar automap |
| `menuactive` | `boolean` | `true` when any menu is visible |
| `itemOn` | `short` | Index of currently selected menu item |
| `skullAnimCounter` | `short` | Countdown for skull cursor animation (8 tics per frame) |
| `whichSkull` | `short` | 0 or 1; selects `M_SKULL1` or `M_SKULL2` patch |
| `currentMenu` | `menu_t *` | Pointer to the currently displayed `menu_t` |
| `skullName[2][9]` | `char[][]` | Names of skull patches: `"M_SKULL1"`, `"M_SKULL2"` |
| `endstring[160]` | `char[]` | Buffer for the quit confirmation message |
| `tempstring[80]` | `char[]` | Buffer for quicksave/quickload confirmation messages |
| `gammamsg[5][26]` | `char[][]` | Gamma level messages (from `dstrings.h`: `GAMMALVL0`..`GAMMALVL4`) |
| `detailNames[2][9]` | `char[][]` | Patch names for detail level indicator: `"M_GDHIGH"`, `"M_GDLOW"` |
| `msgNames[2][9]` | `char[][]` | Patch names for messages on/off: `"M_MSGOFF"`, `"M_MSGON"` |
| `epi` | `int` | Episode chosen in episode selection, stored for passing to `G_DeferedInitNew` |

### External references used

| Variable | Source | Purpose |
|---|---|---|
| `hu_font[HU_FONTSIZE]` | `hu_stuff.c` | Font patches for `M_WriteText` |
| `message_dontfuckwithme` | `hu_stuff.c` | Forces HUD message display |
| `chat_on` | `hu_stuff.c` | Suppress screen resize when chat is active |
| `sendpause` | `g_game.c` | Pause the game |

### Quit sounds

```c
int quitsounds[8]  = { sfx_pldeth, sfx_dmpain, sfx_popain, sfx_slop, sfx_telept, sfx_posit1, sfx_posit3, sfx_sgtatk };
int quitsounds2[8] = { sfx_vilact, sfx_getpow, sfx_boscub, sfx_slop, sfx_skeswg, sfx_kntdth, sfx_bspact, sfx_sgtatk };
```
Random quit sounds played before exit. `quitsounds` is used for shareware/registered/retail; `quitsounds2` for commercial (Doom 2). The index is `(gametic >> 2) & 7`.

---

## Menu Definitions (static data)

The file defines eight complete menus as static arrays of `menuitem_t` and `menu_t` structs:

### Main Menu (`MainMenu` / `MainDef`)
- Items: New Game (n), Options (o), Load Game (l), Save Game (s), Read This (r), Quit DOOM (q)
- Position: x=97, y=64
- In commercial mode `M_Init` removes "Read This!" and shifts items up 8 pixels.

### Episode Select (`EpisodeMenu` / `EpiDef`)
- Items: Knee-Deep in the Dead (k), The Shores of Hell (t), Inferno (i), Thy Flesh Consumed (t)
- Default: ep1
- In shareware/registered mode `M_Init` removes the 4th episode.

### New Game / Skill (`NewGameMenu` / `NewDef`)
- Items: I'm Too Young to Die (i), Hey Not Too Rough (h), Hurt Me Plenty (h), Ultra-Violence (u), Nightmare! (n)
- Default selection: hurtme
- Nightmare triggers a confirmation dialog.

### Options (`OptionsMenu` / `OptionsDef`)
- Items: End Game (1), Messages toggle (1), Detail toggle (1), Screen Size slider (2), [spacer], Mouse Sensitivity slider (2), [spacer], Sound Volume (1)
- Status-2 items use left/right arrow keys.

### Sound Volume (`SoundMenu` / `SoundDef`)
- Items: Sfx Volume slider (2), [spacer], Music Volume slider (2), [spacer]
- Range: 0-15 for each

### Read This page 1 / page 2 (`ReadMenu1`/`ReadDef1`, `ReadMenu2`/`ReadDef2`)
- Single invisible item leading to the next page or back to main.
- Draw routines show `HELP1`/`HELP2`/`CREDIT` depending on game mode.

### Load Game / Save Game (`LoadMenu`/`LoadDef`, `SaveMenu`/`SaveDef`)
- 6 slots each. Item names are empty strings; the save description text is drawn by the menu's draw routine using `savegamestrings[]`.

---

## Functions

### `M_ReadSaveStrings`
```c
void M_ReadSaveStrings(void)
```
**Purpose:** Read the first 24 bytes (the description string) from each of the 6 save game files into `savegamestrings[]`. Called before displaying load or save menus.

**Logic:** Iterates slots 0-5. For each slot, constructs the filename (`doomsav0.dsg` through `doomsav5.dsg`, or `c:\doomdata\doomsavN.dsg` on `-cdrom`). If the file does not exist, stores `EMPTYSTRING` and sets `LoadMenu[i].status = 0` (not selectable). If it exists, reads 24 bytes and sets `status = 1`.

---

### `M_DrawLoad`
```c
void M_DrawLoad(void)
```
**Purpose:** Draw the Load Game menu title and save slot borders with their description strings.

**Logic:** Draws the `M_LOADG` title patch at (72,28). Draws 6 bordered slots using `M_DrawSaveLoadBorder` and `M_WriteText`.

---

### `M_DrawSaveLoadBorder`
```c
void M_DrawSaveLoadBorder(int x, int y)
```
**Purpose:** Draw the decorative border around a save/load text entry field.

**Logic:** Draws a left cap (`M_LSLEFT`), 24 repetitions of a center tile (`M_LSCNTR`) at 8-pixel increments, and a right cap (`M_LSRGHT`). The 24-tile center accommodates 24 characters at 8 pixels each = 192 pixels wide.

---

### `M_LoadSelect`
```c
void M_LoadSelect(int choice)
```
**Purpose:** Load the save game in slot `choice` and close all menus.

---

### `M_LoadGame`
```c
void M_LoadGame(int choice)
```
**Purpose:** Open the Load Game menu. Refuses (displays error message) if a net game is active.

---

### `M_DrawSave`
```c
void M_DrawSave(void)
```
**Purpose:** Draw the Save Game menu. Identical to `M_DrawLoad` with `M_SAVEG` title. If the user is actively typing a name (`saveStringEnter`), draws a `_` cursor after the current text.

---

### `M_DoSave`
```c
void M_DoSave(int slot)
```
**Purpose:** Actually trigger the save. Calls `G_SaveGame`, closes menus, and sets `quickSaveSlot` if it was waiting to be assigned (`-2`).

---

### `M_SaveSelect`
```c
void M_SaveSelect(int choice)
```
**Purpose:** Activate text entry mode for the save string of `choice`. Saves the old string for cancel-recovery, clears the string if it was `EMPTYSTRING`, positions the cursor at the end.

---

### `M_SaveGame`
```c
void M_SaveGame(int choice)
```
**Purpose:** Open the Save Game menu from the main menu. Refused if not currently in a game (`!usergame`) or not at a level.

---

### `M_QuickSaveResponse` / `M_QuickSave`
```c
void M_QuickSaveResponse(int ch)
void M_QuickSave(void)
```
**Purpose:** `M_QuickSave` implements F6 quicksave. If no slot is selected yet, opens the save menu to let the user pick one. If a slot is already selected, prompts with a confirmation message; `M_QuickSaveResponse` performs the actual save if the user answers 'y'.

---

### `M_QuickLoadResponse` / `M_QuickLoad`
```c
void M_QuickLoadResponse(int ch)
void M_QuickLoad(void)
```
**Purpose:** F9 quickload. Blocked in net games. If no slot is set, shows an error. Otherwise confirms and calls `M_LoadSelect`.

---

### `M_DrawReadThis1` / `M_DrawReadThis2`
```c
void M_DrawReadThis1(void)
void M_DrawReadThis2(void)
```
**Purpose:** Draw the help screens. `M_DrawReadThis1` shows `HELP` (commercial), `HELP1` (shareware/registered/retail). `M_DrawReadThis2` shows `CREDIT` (retail/commercial) or `HELP2` (shareware/registered). Both set `inhelpscreens = true`.

---

### `M_DrawSound`
```c
void M_DrawSound(void)
```
**Purpose:** Draw the Sound Volume menu: the `M_SVOL` title patch plus two thermometer sliders for SFX and music volumes, each 16 steps wide.

---

### `M_Sound` / `M_SfxVol` / `M_MusicVol`
```c
void M_Sound(int choice)
void M_SfxVol(int choice)
void M_MusicVol(int choice)
```
**Purpose:** `M_Sound` transitions to the Sound menu. `M_SfxVol` and `M_MusicVol` handle left (0) / right (1) arrows on their respective sliders, clamping volumes to 0-15 and calling `S_SetSfxVolume` / `S_SetMusicVolume`.

---

### `M_DrawMainMenu`
```c
void M_DrawMainMenu(void)
```
**Purpose:** Draw the `M_DOOM` title graphic at position (94, 2).

---

### `M_DrawNewGame` / `M_NewGame` / `M_ChooseSkill` / `M_Episode`
```c
void M_DrawNewGame(void)
void M_NewGame(int choice)
void M_ChooseSkill(int choice)
void M_Episode(int choice)
```
**Purpose:** New Game flow. `M_NewGame` navigates to the episode select (or directly to skill select in commercial mode). `M_Episode` stores the chosen episode in `epi` and navigates to skill select; it enforces shareware/registered restrictions. `M_ChooseSkill` calls `G_DeferedInitNew` with the chosen skill and episode, or displays a Nightmare! confirmation first. `M_DrawNewGame` draws the `M_NEWG` and `M_SKILL` title patches.

---

### `M_DrawOptions` / `M_Options`
```c
void M_DrawOptions(void)
void M_Options(int choice)
```
**Purpose:** `M_Options` opens the Options menu. `M_DrawOptions` draws the `M_OPTTTL` title, the current detail level patch, the current messages on/off patch, the mouse sensitivity thermometer (10 steps), and the screen size thermometer (9 steps).

---

### `M_ChangeMessages`
```c
void M_ChangeMessages(int choice)
```
**Purpose:** Toggle `showMessages` between 0 and 1 and post a confirmation message to the player's HUD. Sets `message_dontfuckwithme = true` to force the HUD to display it even when messages are being turned off.

---

### `M_EndGameResponse` / `M_EndGame`
```c
void M_EndGameResponse(int ch)
void M_EndGame(int choice)
```
**Purpose:** End the current game and return to the title screen. Blocked if no game is active or if it is a net game. Confirmation dialog; on 'y' calls `D_StartTitle`.

---

### `M_ReadThis` / `M_ReadThis2` / `M_FinishReadThis`
```c
void M_ReadThis(int choice)
void M_ReadThis2(int choice)
void M_FinishReadThis(int choice)
```
**Purpose:** Navigate to help page 1, help page 2, and back to the main menu respectively.

---

### `M_QuitResponse` / `M_QuitDOOM`
```c
void M_QuitResponse(int ch)
void M_QuitDOOM(int choice)
```
**Purpose:** Quit the game. `M_QuitDOOM` picks a random quit message from the `endmsg[]` array in `dstrings.h` and shows a confirmation. `M_QuitResponse` on 'y' plays a random quit sound (from `quitsounds` or `quitsounds2`), waits 3 seconds via `I_WaitVBL(105)`, then calls `I_Quit`.

---

### `M_ChangeSensitivity`
```c
void M_ChangeSensitivity(int choice)
```
**Purpose:** Adjust `mouseSensitivity` up or down, clamped to 0-9.

---

### `M_ChangeDetail`
```c
void M_ChangeDetail(int choice)
```
**Purpose:** Toggle `detailLevel` between 0 and 1. **Note:** The actual `R_SetViewSize` call is commented out with a "FIXME" comment; low detail mode was non-functional in this release.

---

### `M_SizeDisplay`
```c
void M_SizeDisplay(int choice)
```
**Purpose:** Adjust screen size. Decrements/increments both `screenSize` (slider display, 0-8) and `screenblocks` (renderer value, 3-11 which is `screenSize + 3`), then calls `R_SetViewSize`.

---

### `M_DrawThermo`
```c
void M_DrawThermo(int x, int y, int thermWidth, int thermDot)
```
**Purpose:** Draw a thermometer/slider widget at (x, y). The slider consists of a left cap (`M_THERML`), `thermWidth` center tiles (`M_THERMM`), a right cap (`M_THERMR`), and a filled dot indicator (`M_THERMO`) positioned at `thermDot` steps from the left. Each step is 8 pixels wide.

**Parameters:**
- `x`, `y` - Screen coordinates of the left edge.
- `thermWidth` - Number of steps/divisions in the slider.
- `thermDot` - Current value (0-based), where the fill indicator is drawn.

---

### `M_DrawEmptyCell` / `M_DrawSelCell`
```c
void M_DrawEmptyCell(menu_t *menu, int item)
void M_DrawSelCell(menu_t *menu, int item)
```
**Purpose:** Draw the empty (`M_CELL1`) or selected (`M_CELL2`) cell indicator for an item. Used in NetGame menus (the standard skull cursor is used instead in single-player menus).

---

### `M_StartMessage`
```c
void M_StartMessage(char *string, void *routine, boolean input)
```
**Purpose:** Display a modal message overlay. Saves current menu state, sets `messageToPrint = 1`, stores the message text pointer and response callback, and activates the menu if not already active.

**Parameters:**
- `string` - Message text (may contain `\n` newlines).
- `routine` - Callback `void (*)(int ch)` called with the key pressed; `NULL` for info-only messages.
- `input` - `true` if the message requires a y/n/Escape response; `false` for timed/info messages that dismiss on any key.

---

### `M_StopMessage`
```c
void M_StopMessage(void)
```
**Purpose:** Dismiss the message overlay and restore the previous `menuactive` state.

---

### `M_StringWidth`
```c
int M_StringWidth(char *string)
```
**Purpose:** Calculate the pixel width of a string rendered in `hu_font`. Non-printable characters contribute 4 pixels. Used to center message text and position save-string cursors.

---

### `M_StringHeight`
```c
int M_StringHeight(char *string)
```
**Purpose:** Calculate the pixel height of a (possibly multi-line) string rendered in `hu_font`. Each `\n` adds one line's height.

---

### `M_WriteText`
```c
void M_WriteText(int x, int y, char *string)
```
**Purpose:** Render a string at (x, y) using `hu_font` patches via `V_DrawPatchDirect`. Handles newlines (advances `cy` by 12 pixels, resets `cx` to `x`). Stops if the line would exceed `SCREENWIDTH`. Non-printable characters advance `cx` by 4 pixels.

---

### `M_Responder`
```c
boolean M_Responder(event_t *ev)
```
**Purpose:** Main event handler for all menu input. Returns `true` if the event was consumed.

**Event source handling:**
- **Joystick:** Maps axis-3 to up/down, axis-2 to left/right (with a 5-tic rate limiter), button-1 to Enter, button-2 to Backspace.
- **Mouse:** Maps Y-axis delta > 30 to up/down, X-axis delta > 30 to left/right (rate limited), left button to Enter, right button to Backspace.
- **Keyboard:** Passes `ev->data1` directly as `ch`.

**Processing order (highest priority first):**
1. Save-game string input mode (`saveStringEnter`): Handles Backspace, Escape (cancel), Enter (confirm), and printable characters (appended if within `SAVESTRINGSIZE-1` and the rendered width fits).
2. Message-awaiting-input mode (`messageToPrint`): Dismisses on any key; calls `messageRoutine(ch)` if set.
3. F-key shortcuts (when menu not active): F1 help, F2 save, F3 load, F4 sound, F5 detail, F6 quicksave, F7 end game, F8 messages, F9 quickload, F10 quit, F11 gamma, +/- screen resize.
4. Escape when menu not active: Opens main menu.
5. Navigation within active menu: Down/Up (skip status=-1 items), Left/Right (activate slider items with choice 0/1), Enter (activate item), Escape (clear all menus), Backspace (go to `prevMenu`), alpha-key hotkey scan.

---

### `M_StartControlPanel`
```c
void M_StartControlPanel(void)
```
**Purpose:** Open the main menu if not already open. Sets `menuactive = 1`, `currentMenu = &MainDef`, restores `itemOn`.

---

### `M_Drawer`
```c
void M_Drawer(void)
```
**Purpose:** Render the menu system to the screen buffer each frame (called after the 3D view).

**Logic:**
1. Clears `inhelpscreens`.
2. If `messageToPrint`: Computes vertical center from `M_StringHeight`, splits the message on `\n`, horizontally centers each line via `M_StringWidth`, and draws with `M_WriteText`. Returns immediately.
3. If `!menuactive`: Returns.
4. Calls `currentMenu->routine()` to draw the menu's title/background graphic.
5. Iterates `currentMenu->numitems`, drawing each non-empty `name` patch at `(x, y + i*LINEHEIGHT)`.
6. Draws the animated skull cursor at `(x + SKULLXOFF, y - 5 + itemOn*LINEHEIGHT)`.

`SKULLXOFF` is `-32` (the skull is drawn 32 pixels to the left of the item graphics). `LINEHEIGHT` is `16` pixels between items.

---

### `M_ClearMenus`
```c
void M_ClearMenus(void)
```
**Purpose:** Close all menus by setting `menuactive = 0`.

---

### `M_SetupNextMenu`
```c
void M_SetupNextMenu(menu_t *menudef)
```
**Purpose:** Navigate to a different menu. Sets `currentMenu = menudef` and restores `itemOn` to the menu's `lastOn`.

---

### `M_Ticker`
```c
void M_Ticker(void)
```
**Purpose:** Called every game tic. Decrements `skullAnimCounter`; when it reaches 0, toggles `whichSkull` (0 XOR 1) and resets the counter to 8. This animates the skull cursor at approximately 4.4 Hz.

---

### `M_Init`
```c
void M_Init(void)
```
**Purpose:** Initialize the menu system at game startup.

**Logic:**
- Sets `currentMenu = &MainDef`, `menuactive = 0`, skull animation state.
- Derives `screenSize = screenblocks - 3`.
- Clears message state, sets `quickSaveSlot = -1`.
- Performs game-mode-specific adjustments:
  - **commercial:** Replaces "Read This!" main menu item with "Quit DOOM", reduces `MainDef.numitems` by 1, shifts menu down 8 pixels, updates `NewDef.prevMenu`, adjusts `ReadDef1` to use `M_FinishReadThis` and repositions it.
  - **shareware/registered:** Removes 4th episode by decrementing `EpiDef.numitems`.
  - **retail:** No adjustments.

---

## Dependencies

| Header / Module | Purpose |
|---|---|
| `doomdef.h` | `boolean`, game mode constants, screen dimensions |
| `dstrings.h` | String constants for menus, quit messages, `endmsg[]` |
| `d_main.h` | `D_StartTitle` |
| `i_system.h` | `I_Quit`, `I_WaitVBL`, `I_GetTime` |
| `i_video.h` | `I_SetPalette` (gamma change) |
| `z_zone.h` | `PU_CACHE` zone tag |
| `v_video.h` | `V_DrawPatchDirect` |
| `w_wad.h` | `W_CacheLumpName` |
| `r_local.h` | `R_SetViewSize` |
| `hu_stuff.h` | `hu_font`, `HU_FONTSIZE`, `HU_FONTSTART`, `message_dontfuckwithme`, `chat_on` |
| `g_game.h` | `G_SaveGame`, `G_LoadGame`, `G_DeferedInitNew`, `G_ScreenShot`, `gamestate`, `usergame`, `netgame`, `demoplayback` |
| `m_argv.h` | `M_CheckParm("-cdrom")` for CD-ROM save path |
| `m_swap.h` | `SHORT` macro for endian-safe patch width/height reads |
| `s_sound.h` | `S_StartSound`, `S_SetSfxVolume`, `S_SetMusicVolume` |
| `doomstat.h` | `gamemode`, `gametic`, `consoleplayer`, `players[]`, `automapactive`, `devparm`, `usegamma`, `snd_SfxVolume`, `snd_MusicVolume`, `language` |
| `sounds.h` | `sfx_*` enum values for quit sounds and UI sounds |
| `m_menu.h` | Own public interface |
