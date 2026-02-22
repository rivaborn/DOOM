# File Overview

`hu_stuff.h` is the public interface header for DOOM's high-level heads-up display system. It defines the font range constants, layout constants, and the complete set of public functions for initializing, ticking, drawing, erasing, and responding to input for the HUD. Any module that needs to interact with the HUD — including the game loop (`d_main.c`), the networking layer, and the status bar — includes this header.

## Global Variables

None are declared in this header. All globals are defined in `hu_stuff.c`.

## Constants and Macros

### Font Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `HU_FONTSTART` | `'!'` (33) | ASCII code of the first character in the HUD font (`STCFN033`). |
| `HU_FONTEND` | `'_'` (95) | ASCII code of the last character in the HUD font (`STCFN095`). |
| `HU_FONTSIZE` | `(HU_FONTEND - HU_FONTSTART + 1)` = 63 | Total number of glyphs in the HUD font. |

The HUD font covers printable ASCII from `!` (33) to `_` (95), including all uppercase letters, digits, and common punctuation. Lowercase letters are not directly supported; text is uppercased before rendering.

### Chat and Broadcast Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `HU_BROADCAST` | `5` | Special destination code meaning "send to all players." Values 1–4 are individual player numbers. |

### Message Display Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `HU_MSGREFRESH` | `KEY_ENTER` | Key that re-displays the last message. |
| `HU_MSGX` | `0` | X screen position of the message display area. |
| `HU_MSGY` | `0` | Y screen position of the message display area. |
| `HU_MSGWIDTH` | `64` | Width of the message area in characters. |
| `HU_MSGHEIGHT` | `1` | Height of the message area in lines. |
| `HU_MSGTIMEOUT` | `(4*TICRATE)` | Duration a message is displayed (4 seconds at the engine's tic rate). |

## Functions

### `HU_Init`

```c
void HU_Init(void);
```

Called once at program startup by `D_DoomMain`. Loads the HUD font patches from the WAD and sets up the keyboard shift transform table (English or French). Must be called before any widget operations.

---

### `HU_Start`

```c
void HU_Start(void);
```

Called at the start of each map by the game loop. Constructs all HUD widgets (message display, map title, chat input, per-player input buffers) for the current level.

---

### `HU_Responder`

```c
boolean HU_Responder(event_t* ev);
```

Called by the event dispatch system for every input event. Handles chat mode activation, player targeting in multiplayer, character input, macro sending, and message refresh. Returns `true` if the event was consumed.

---

### `HU_Ticker`

```c
void HU_Ticker(void);
```

Called once per game tic. Updates the message timeout counter, shows new player messages, and processes incoming chat characters from remote players in multiplayer sessions.

---

### `HU_Drawer`

```c
void HU_Drawer(void);
```

Called every frame to draw all active HUD widgets (messages, chat input, map title) to the screen.

---

### `HU_dequeueChatChar`

```c
char HU_dequeueChatChar(void);
```

Called by the networking layer (`d_net.c` / `g_game.c`) to retrieve the next outgoing chat character for transmission. Returns `0` when the queue is empty.

---

### `HU_Erase`

```c
void HU_Erase(void);
```

Called every frame to erase HUD widget backgrounds before redrawing, ensuring clean display in reduced screen size mode where borders are visible.

## Dependencies

- `d_event.h` — `event_t` type required by `HU_Responder`.
