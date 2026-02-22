# File Overview

`hu_stuff.c` implements the high-level heads-up display (HUD) system for DOOM. It sits above the widget library (`hu_lib.c`) and ties together the game's player message display, the automap level name title, and the multiplayer chat system. This module is responsible for loading the HUD font from the WAD, constructing the HUD widgets at map start, processing keyboard input for chat mode, routing incoming chat messages from other players, and managing the message timeout timer. It also contains all map name string tables for DOOM, DOOM II, and the Final DOOM episodes (The Plutonia Experiment and TNT: Evilution), as well as the keyboard shift-transform tables for both English and French keyboards.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `char*` | `chat_macros[10]` | Array of 10 predefined chat macro strings (`HUSTR_CHATMACRO0`–`HUSTR_CHATMACRO9`). Bound to Alt+0 through Alt+9 during chat. |
| `char*` | `player_names[4]` | Display names for the four players: `"GREEN: "`, `"INDIGO: "`, `"BROWN: "`, `"RED: "`. Prepended to incoming chat messages. |
| `char` | `chat_char` | The last character enqueued for chat transmission. Described as "remove later." |
| `patch_t*` | `hu_font[HU_FONTSIZE]` | Array of pointers to the 59 font glyph patches (`STCFN033`–`STCFN095`) loaded from the WAD. This is the shared HUD font used by menus, status bar, and HUD widgets. |
| `boolean` | `chat_on` | `true` when the player has the chat input box open. |
| `boolean` | `message_dontfuckwithme` | When `true`, forces the current message to display even if another message is already showing. Set by critical game messages. |
| `char*` | `mapnames[]` | 45-entry array of map title strings for DOOM shareware/registered/retail (Ultimate). Covers episodes 1–4 (E1M1–E4M9) plus 9 placeholder `"NEWLEVEL"` entries. |
| `char*` | `mapnames2[]` | 32-entry array of map title strings for DOOM II (MAP01–MAP32). |
| `char*` | `mapnamesp[]` | 32-entry array for The Plutonia Experiment map titles. |
| `char*` | `mapnamest[]` | 32-entry array for TNT: Evilution map titles. |
| `const char*` | `shiftxform` | Pointer to the active shift-key transform table (set to `english_shiftxform` or `french_shiftxform` at init). |
| `const char` | `french_shiftxform[128]` | Shift key transformation table for French keyboards (AZERTY layout). |
| `const char` | `english_shiftxform[128]` | Shift key transformation table for English keyboards (QWERTY layout). Maps Shift+key to the corresponding shifted character. |
| `char` | `frenchKeyMap[128]` | Character mapping table that translates AZERTY key positions to QWERTY equivalents for French keyboards. |

### Static Variables

| Type | Name | Description |
|------|------|-------------|
| `static player_t*` | `plr` | Pointer to the console player's `player_t`. Set in `HU_Start`. |
| `static hu_textline_t` | `w_title` | The map title text line widget shown in the automap. |
| `static hu_itext_t` | `w_chat` | The chat input widget. |
| `static boolean` | `always_off` | A permanent `false` value used as the `on` pointer for inactive `w_inputbuffer` widgets. |
| `static char` | `chat_dest[MAXPLAYERS]` | Per-player chat destination code (broadcast or target player number). |
| `static hu_itext_t` | `w_inputbuffer[MAXPLAYERS]` | Per-player input buffers for assembling incoming chat messages in multiplayer. |
| `static boolean` | `message_on` | `true` when a player message is currently being displayed. |
| `static boolean` | `message_nottobefuckedwith` | When `true`, suppresses new messages from overriding the current one (set when a `message_dontfuckwithme` message is showing). |
| `static hu_stext_t` | `w_message` | The scrolling message display widget (upper-left corner). |
| `static int` | `message_counter` | Countdown timer (in tics) until the current message disappears. |
| `static boolean` | `headsupactive` | `true` after `HU_Start` has been called; `false` after `HU_Stop`. |
| `static char` | `chatchars[128]` | Circular queue buffer for outgoing chat characters. |
| `static int` | `head` | Write index into `chatchars`. |
| `static int` | `tail` | Read index into `chatchars`. |

## Functions

### `ForeignTranslation`

```c
char ForeignTranslation(unsigned char ch)
```

**Purpose:** Translates an AZERTY key code to its QWERTY equivalent for French keyboard support.

**Parameters:**
- `ch` — Raw key code.

**Returns:** Translated character if `ch < 128`, otherwise `ch` unchanged.

---

### `HU_Init`

```c
void HU_Init(void)
```

**Purpose:** Initializes the HUD system at program startup. Sets the active shift transform table based on the `french` flag, and loads all 59 HUD font patches (`STCFN033` through `STCFN095`) from the WAD into `hu_font[]`.

**Parameters:** None.
**Returns:** Nothing.

**Key logic:** Iterates `i` from 0 to `HU_FONTSIZE-1`, building names `"STCFN033"` through `"STCFN095"` and caching each with `W_CacheLumpName(..., PU_STATIC)`.

---

### `HU_Stop`

```c
void HU_Stop(void)
```

**Purpose:** Marks the HUD as inactive (`headsupactive = false`). Called at the start of `HU_Start` if already active, effectively reinitializing.

**Parameters:** None.
**Returns:** Nothing.

---

### `HU_Start`

```c
void HU_Start(void)
```

**Purpose:** Sets up all HUD widgets for the current level. Creates the message widget, map title widget (populated with the current level name), chat input widget, and per-player input buffer widgets. Called at the start of each map.

**Parameters:** None.
**Returns:** Nothing.

**Key logic:**
1. Calls `HU_Stop` if already active.
2. Sets `plr = &players[consoleplayer]`.
3. Resets `message_on`, `message_dontfuckwithme`, `message_nottobefuckedwith`, `chat_on`.
4. Initializes `w_message` (scrolling text) at `HU_MSGX, HU_MSGY` with height `HU_MSGHEIGHT`.
5. Initializes `w_title` (text line) at `HU_TITLEX, HU_TITLEY`, then selects the correct map name string from `mapnames`, `mapnames2`, `mapnamesp`, or `mapnamest` based on `gamemode`, and adds it character by character.
6. Initializes `w_chat` (input text) for outgoing chat.
7. Initializes `w_inputbuffer[i]` for each player slot (all permanently off via `always_off`).

---

### `HU_Drawer`

```c
void HU_Drawer(void)
```

**Purpose:** Draws all active HUD widgets to the screen. Called every frame by the main game loop.

**Parameters:** None.
**Returns:** Nothing.

**Key logic:** Calls `HUlib_drawSText(&w_message)`, `HUlib_drawIText(&w_chat)`, and — only when the automap is active — `HUlib_drawTextLine(&w_title, false)`.

---

### `HU_Erase`

```c
void HU_Erase(void)
```

**Purpose:** Erases all HUD widgets from the screen (restoring the background). Called every frame before redrawing.

**Parameters:** None.
**Returns:** Nothing.

---

### `HU_Ticker`

```c
void HU_Ticker(void)
```

**Purpose:** Per-tic update for the HUD. Manages the message timeout countdown, displays new player messages, and processes incoming chat characters from other players in multiplayer.

**Parameters:** None.
**Returns:** Nothing.

**Key logic:**
1. If `message_counter > 0`, decrements it; when it reaches zero, clears `message_on` and `message_nottobefuckedwith`.
2. If `showMessages` or `message_dontfuckwithme`, checks `plr->message`: if set, adds it to `w_message`, resets the timer to `HU_MSGTIMEOUT` (4 seconds), and clears `plr->message`.
3. In netgame mode, iterates over all players; for each remote player's `cmd.chatchar`:
   - If the chatchar is a destination code (`<= HU_BROADCAST`), stores it in `chat_dest[i]`.
   - Otherwise, feeds the character through the shift transform (for lowercase letters) and into `w_inputbuffer[i]` via `HUlib_keyInIText`.
   - If Enter is received and the message is addressed to the console player or broadcast, displays it as a player message with the sender's name.
   - Plays `sfx_radio` (DOOM II) or `sfx_tink` (DOOM I) as a notification sound.

---

### `HU_queueChatChar`

```c
void HU_queueChatChar(char c)
```

**Purpose:** Enqueues a character into the outgoing chat circular buffer (`chatchars`). If the buffer is full, sets `plr->message` to the "message unsent" string.

**Parameters:**
- `c` — Character to enqueue.

**Returns:** Nothing.

---

### `HU_dequeueChatChar`

```c
char HU_dequeueChatChar(void)
```

**Purpose:** Dequeues and returns the next character from the outgoing chat circular buffer. Returns 0 if the buffer is empty. Called by the networking code to transmit chat input to other players.

**Parameters:** None.
**Returns:** Next queued character, or `0` if the queue is empty.

---

### `HU_Responder`

```c
boolean HU_Responder(event_t *ev)
```

**Purpose:** Handles keyboard input for the HUD system. Manages shift/alt state tracking, message refresh, chat mode activation, player-to-player targeting, chat macro sending, and character input into the chat box.

**Parameters:**
- `ev` — Pointer to the input event to process.

**Returns:** `true` if the event was consumed (should not be processed further); `false` otherwise.

**Key logic:**
- Tracks `shiftdown` and `altdown` state for modifier keys (Shift, Alt).
- Only processes `ev_keydown` events for all non-modifier logic.
- **Out of chat mode:**
  - `HU_MSGREFRESH` (Enter): refreshes the current message display.
  - `HU_INPUTTOGGLE` ('t'): activates broadcast chat mode.
  - In netgame with 3+ players, pressing a destination key (G/I/B/R for green/indigo/brown/red) starts a targeted chat. Pressing your own key shows progressively snarky "talk to yourself" messages.
- **In chat mode:**
  - Alt+digit: sends the corresponding `chat_macros[]` string via `HU_queueChatChar`.
  - Regular characters: passed through French translation if needed, then through `shiftxform` if Shift is held, then fed to `w_chat` via `HUlib_keyInIText` and queued.
  - Enter: closes chat and copies the typed message to `plr->message` for display.
  - Escape: closes chat without sending.

## Data Structures

### `menuitem_t` / `menu_t`

Not defined here; see `m_menu.h`.

### Key Constants (from this file's context)

| Constant | Value | Description |
|----------|-------|-------------|
| `HU_TITLE` | `mapnames[(gameepisode-1)*9+gamemap-1]` | Current map title for DOOM episodes. |
| `HU_TITLE2` | `mapnames2[gamemap-1]` | Current map title for DOOM II. |
| `HU_TITLEP` | `mapnamesp[gamemap-1]` | Current map title for Plutonia. |
| `HU_TITLET` | `mapnamest[gamemap-1]` | Current map title for TNT. |
| `HU_TITLEY` | `167 - SHORT(hu_font[0]->height)` | Y position of the map title (just above the status bar). |
| `HU_INPUTTOGGLE` | `'t'` | Key to activate chat mode. |
| `QUEUESIZE` | `128` | Size of the outgoing chat character queue. |

## Dependencies

- `doomdef.h` — Core types, `MAXPLAYERS`, game mode constants.
- `z_zone.h` — Memory management (indirectly through WAD loading).
- `m_swap.h` — `SHORT` macro for patch dimensions.
- `hu_stuff.h` — Own header.
- `hu_lib.h` — Widget library functions and types.
- `w_wad.h` — `W_CacheLumpName` for loading font patches.
- `s_sound.h` — `S_StartSound` for chat notification sounds.
- `doomstat.h` — `netgame`, `consoleplayer`, `players[]`, `playeringame[]`, `gamemode`, `gameepisode`, `gamemap`, `french`.
- `dstrings.h` — All string constants (`HUSTR_*`, `HUSTR_CHATMACRO*`, map names).
- `sounds.h` — Sound effect IDs (`sfx_radio`, `sfx_tink`).
- `d_event.h` — `event_t` type.
