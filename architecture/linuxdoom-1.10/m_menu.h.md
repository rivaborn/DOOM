# File Overview

`m_menu.h` is the public interface header for DOOM's in-game menu system. It exposes only the five functions that the main game loop and initialization code need to call; all internal menu data structures, menu definitions, and helper functions are kept private inside `m_menu.c`.

The menu system handles:
- The main pause/escape menu hierarchy (New Game, Options, Load, Save, etc.)
- Options sub-menus (sound volumes, screen size, mouse sensitivity, detail level)
- Save/load game string entry
- Modal message boxes (confirmation dialogs, shareware promotion screens)
- Hot-key F-key bindings (F1 help, F2 save, F3 load, F4 sound, F6 quicksave, etc.)

---

## Global Variables

None declared in this header. All menu state variables are static or file-scope in `m_menu.c`.

---

## Functions

### `M_Responder`
```c
boolean M_Responder(event_t *ev);
```
**Purpose:** Process a single input event on behalf of the menu system. This is called by the main game loop for every event (`ev_keydown`, `ev_mouse`, `ev_joystick`) before the game logic sees it.

**Parameters:**
- `ev` (`event_t *`) - Pointer to the input event to process.

**Returns:** `boolean` - `true` if the event was consumed by the menu (the game loop should not process it further); `false` if the event was not handled.

**Role:** Even when no menu is visible, `M_Responder` handles F-key shortcuts (quicksave, quickload, screenshot, gamma, screen resize) and opens the menu on Escape. When a menu is active, it routes navigation keys (up/down/left/right/enter/backspace/escape) to the appropriate menu actions. It also processes save-game string typing when the user is entering a save slot name.

---

### `M_Ticker`
```c
void M_Ticker(void);
```
**Purpose:** Advance menu animation by one game tic. Called once per game tic by the main loop.

**Role:** Drives the animated skull cursor. Every 8 tics the skull alternates between its two frames (`M_SKULL1` / `M_SKULL2`), creating the blinking/wiggling effect seen beside the selected menu item.

---

### `M_Drawer`
```c
void M_Drawer(void);
```
**Purpose:** Render the currently active menu (or message box) directly into the screen buffer. Called after the 3D view has been rendered but before the final blit to the display.

**Role:** If a message box is active, centers and draws the message string using `hu_font`. Otherwise, if the menu is active, calls the current menu's draw routine (to render the menu title graphic), then draws each menu item's patch graphic at the appropriate Y position, and finally draws the animated skull cursor beside the selected item. All drawing is done via `V_DrawPatchDirect`.

---

### `M_Init`
```c
void M_Init(void);
```
**Purpose:** Initialize the menu system at startup. Called by `D_DoomMain` during game initialization, after the WAD is loaded.

**Role:** Sets up initial menu state (points `currentMenu` to `MainDef`, clears `menuactive`, initializes skull animation counters, reads `screenblocks` into `screenSize`). Also performs game-mode-specific menu adjustments:
- **commercial (Doom 2):** Removes the "Read This!" item from the main menu (Doom 2 had only one help page), shifts the remaining items up, and adjusts `ReadDef1` to use `M_FinishReadThis` as its action.
- **shareware/registered:** Removes the 4th episode from the episode selection menu.
- **retail:** No changes; all four episodes are available.

---

### `M_StartControlPanel`
```c
void M_StartControlPanel(void);
```
**Purpose:** Open the main menu if it is not already open. Called by the intro/demo loop when the player presses a key during the title screen.

**Role:** Sets `menuactive = 1`, points `currentMenu` to `MainDef`, and restores the last selected item position. If the menu is already active this function does nothing (the check prevents recursive activation).

---

## Dependencies

- `d_event.h` - Defines `event_t` and `ev_keydown`, `ev_mouse`, `ev_joystick` event type constants needed by `M_Responder`.
