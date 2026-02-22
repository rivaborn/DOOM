# File Overview

`d_event.h` defines the fundamental event and input system data structures for DOOM. It describes how raw hardware input (keyboard, mouse, joystick) is packaged into generic `event_t` records that are queued and dispatched to the various subsystem responder functions each tic. It also defines the `gameaction_t` enum that drives deferred state transitions (load level, save game, etc.) and the `buttoncode_t` enum that encodes in-game commands (fire, use, weapon change, pause, save) inside a single byte for network transmission.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `event_t[MAXEVENTS]` | `events` | Circular buffer holding up to 64 pending input events. |
| `int` | `eventhead` | Write index into `events[]`; advanced by `D_PostEvent`. |
| `int` | `eventtail` | Read index into `events[]`; advanced by `D_ProcessEvents`. |
| `gameaction_t` | `gameaction` | Current pending deferred action; set by game code and consumed by `G_Ticker`. |

(`MAXEVENTS = 64`)

---

## Functions

None defined in this file.

---

## Data Structures

### `evtype_t`
```c
typedef enum {
    ev_keydown,
    ev_joystick,
    ev_mouse,
    ev_joystick
} evtype_t;
```
Classifies the source and nature of an event:
- `ev_keydown` — a key (or button) was pressed.
- `ev_keyup` — a key (or button) was released.
- `ev_mouse` — mouse movement or button state update.
- `ev_joystick` — joystick axis/button state update.

---

### `event_t`
```c
typedef struct {
    evtype_t  type;
    int       data1;   // keys / mouse/joystick buttons
    int       data2;   // mouse/joystick x move
    int       data3;   // mouse/joystick y move
} event_t;
```
Generic input event record. The interpretation of `data1`–`data3` depends on `type`:

| `type` | `data1` | `data2` | `data3` |
|--------|---------|---------|---------|
| `ev_keydown` / `ev_keyup` | Key code (ASCII or `KEY_*` constant) | unused | unused |
| `ev_mouse` | Button bitmask | X delta | Y delta |
| `ev_joystick` | Button bitmask | X axis | Y axis |

Special synthetic events (e.g. automap open/close notifications to the status bar) encode a packed integer in `data1` using the `AM_MSGHEADER` scheme defined in `am_map.h`.

---

### `gameaction_t`
```c
typedef enum {
    ga_nothing,
    ga_loadlevel,
    ga_newgame,
    ga_loadgame,
    ga_savegame,
    ga_playdemo,
    ga_completed,
    ga_victory,
    ga_worlddone,
    ga_screenshot
} gameaction_t;
```
An enum used as a one-slot command queue for deferred game-state transitions. Various subsystems (menus, game logic) write a value here, and `G_Ticker` reads and dispatches it at the start of the next tic. `ga_nothing` means no pending action.

---

### `buttoncode_t`
```c
typedef enum {
    BT_ATTACK       = 1,
    BT_USE          = 2,
    BT_SPECIAL      = 128,
    BT_SPECIALMASK  = 3,
    BT_CHANGE       = 4,
    BT_WEAPONMASK   = (8+16+32),
    BT_WEAPONSHIFT  = 3,
    BTS_PAUSE       = 1,
    BTS_SAVEGAME    = 2,
    BTS_SAVEMASK    = (4+8+16),
    BTS_SAVESHIFT   = 2,
} buttoncode_t;
```
Bit-packed encoding for the `buttons` field of `ticcmd_t`. The byte is divided into two independent regions:

**Normal buttons** (low 7 bits):
- Bit 0 (`BT_ATTACK`): fire weapon.
- Bit 1 (`BT_USE`): activate/open.
- Bit 2 (`BT_CHANGE`): weapon change pending; bits 3–5 (`BT_WEAPONMASK >> BT_WEAPONSHIFT`) hold the new weapon number.

**Special commands** (bit 7 set = `BT_SPECIAL`):
- Bits 0–1 (`BT_SPECIALMASK`): special type — `BTS_PAUSE` (1) or `BTS_SAVEGAME` (2).
- Bits 2–4 (`BTS_SAVEMASK`): save-game slot number when `BTS_SAVEGAME`.

This encoding allows a single byte to carry all player actions for network transmission in `ticcmd_t`.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `doomtype.h` | `boolean` type. |
