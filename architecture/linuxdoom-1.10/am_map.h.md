# File Overview

`am_map.h` is the public interface header for the automap module (`am_map.c`). It exposes the four functions that the main game loop calls to integrate the automap into the engine, plus two message code constants used to communicate with the status bar when the automap is opened or closed.

---

## Global Variables

This header declares no global variables directly. The `automapactive` boolean declared in `am_map.c` is referenced via `doomstat.h`.

---

## Functions

### `AM_Responder`
```c
boolean AM_Responder(event_t* ev);
```
Processes input events for the automap. Called by the main game loop's event dispatch chain for every queued event.
- **ev**: Pointer to the event to process.
- **Returns**: `true` if the event was consumed by the automap and should not be forwarded further.

---

### `AM_Ticker`
```c
void AM_Ticker(void);
```
Per-game-tic update function for the automap. Advances the internal clock, applies zoom and pan increments, and updates the follow-player position. Called once per tic by the main game loop.

---

### `AM_Drawer`
```c
void AM_Drawer(void);
```
Renders the automap to the screen framebuffer. Called by the main display function in place of the 3D renderer when the automap is active.

---

### `AM_Stop`
```c
void AM_Stop(void);
```
Forcibly closes the automap. Called when the level completes while the automap is still displayed, ensuring a clean shutdown.

---

## Data Structures

None defined in this header.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `d_event.h` | `event_t` type used by `AM_Responder`. |
| `doomtype.h` | `boolean` type (transitively via `d_event.h`). |

### Message Codes

```c
#define AM_MSGHEADER   (('a'<<24)+('m'<<16))
#define AM_MSGENTERED  (AM_MSGHEADER | ('e'<<8))
#define AM_MSGEXITED   (AM_MSGHEADER | ('x'<<8))
```

These packed integer constants are placed in the `data1` field of a synthetic `ev_keyup` event that `am_map.c` sends to `ST_Responder` (the status bar) to notify it when the automap is opened (`AM_MSGENTERED`) or closed (`AM_MSGEXITED`). The status bar uses these notifications to update its display mode.
