# File Overview

`d_main.h` is the public interface header for the main DOOM startup and game-loop module (`d_main.c`). It exposes the primary entry point, the event-posting function used by the platform layer, and the demo/title-loop helpers that other subsystems can trigger. It also declares the `wadfiles[]` array so other modules can add files to the load list before `W_InitMultipleFiles` is called.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `char*[MAXWADFILES]` | `wadfiles` | Array of WAD file paths to be loaded at startup. `MAXWADFILES = 20`. Modules call `D_AddFile` to append entries rather than writing directly. |

---

## Functions

### `D_AddFile`
```c
void D_AddFile(char* file);
```
Appends a file path to the `wadfiles[]` list for loading by `W_InitMultipleFiles`. Called during startup before WAD initialisation.
- **file**: Path to the WAD or lump file to add.

---

### `D_DoomMain`
```c
void D_DoomMain(void);
```
The top-level engine entry point. Called from `i_main.c` (the platform-specific `main()`). Initialises all subsystems, processes command-line arguments, and enters the main game loop via `D_DoomLoop`. Never returns.

---

### `D_PostEvent`
```c
void D_PostEvent(event_t* ev);
```
Called by I/O functions (keyboard, mouse, joystick handlers in `i_video.c` / `i_system.c`) when input is detected. Enqueues the event in the circular `events[]` buffer for processing by `D_ProcessEvents` during the next tic.
- **ev**: Pointer to the event to enqueue. The event is copied by value.

---

### `D_PageTicker`
```c
void D_PageTicker(void);
```
Per-tic timer callback for title-screen and credit-screen pages. Decrements the page display timer and calls `D_AdvanceDemo` when it expires.

---

### `D_PageDrawer`
```c
void D_PageDrawer(void);
```
Renders the current title or credit page by drawing the `pagename` WAD lump as a full-screen patch.

---

### `D_AdvanceDemo`
```c
void D_AdvanceDemo(void);
```
Signals that the current demo or title screen is finished and the sequence should advance to the next item. Sets an internal flag; the actual advance is performed by `D_DoAdvanceDemo` at the start of the next tic.

---

### `D_StartTitle`
```c
void D_StartTitle(void);
```
Resets the demo sequence counter and starts the title/demo attraction loop. Called when no game has been started yet (e.g. at startup, or after returning from a demo).

---

## Data Structures

None defined in this header.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `d_event.h` | `event_t` type required by `D_PostEvent`. |
