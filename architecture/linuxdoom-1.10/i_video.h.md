# File Overview

`i_video.h` is the platform-independent video interface header for the DOOM engine. It declares the abstract graphics API that the engine's portable code uses to interact with the display system. The implementation of these functions is entirely platform-specific and lives in `i_video.c` (which uses X11 on Linux/UNIX).

This header is analogous to `i_system.h` and `i_sound.h` — it defines the contract between the engine and the platform. Any port of DOOM to a new platform only needs to provide implementations of these functions that match this interface.

The `#pragma interface` directive supports GNU C++ interface/implementation separation.

## Global Variables

No global variables are declared in this header. All video state (X11 display handles, window, colormap, etc.) is managed in the implementation file.

## Functions

### `I_InitGraphics`

**Signature:** `void I_InitGraphics(void)`

**Purpose:** Initializes the graphics hardware or windowing system and sets up the video mode. Called once by `D_DoomMain` during engine startup after all other initialization.

**Parameters:** None.

**Return value:** None.

**Key logic:** Implementation-specific. On Linux/X11, this:
- Opens a connection to the X display server
- Creates an 8-bit PseudoColor window of appropriate size (with optional scale multiplier)
- Sets up the colormap and graphics context
- Optionally uses the MIT-SHM extension for fast screen blitting
- Waits for the window to be ready before returning

After this call, `screens[0]` (the primary framebuffer) is valid and can be drawn into.

---

### `I_ShutdownGraphics`

**Signature:** `void I_ShutdownGraphics(void)`

**Purpose:** Releases all graphics resources and closes the display. Called during engine shutdown sequences (`I_Quit` and `I_Error`).

**Parameters:** None.

**Return value:** None.

**Key logic:** On Linux/X11, detaches from the MIT-SHM extension, releases the shared memory segment, and nullifies the image data pointer.

---

### `I_SetPalette`

**Signature:** `void I_SetPalette(byte* palette)`

**Purpose:** Updates the display's color palette with new RGB values. Called whenever DOOM changes its active palette (e.g., damage flash, invulnerability, sector lighting changes via the `PLAYPAL` WAD lump).

**Parameters:**
- `palette` - Pointer to 768 bytes of raw palette data: 256 color entries, each consisting of three bytes (red, green, blue) with values 0-255. This is "full 8-bit values" as noted in the header comment.

**Return value:** None.

**Key logic:** On Linux/X11, applies the current gamma correction table (`gammatable[usegamma]`) to each color component, converts to 16-bit X11 color values, and uploads all 256 entries to the X11 colormap with `XStoreColors`.

---

### `I_UpdateNoBlit`

**Signature:** `void I_UpdateNoBlit(void)`

**Purpose:** Reserved for updating the screen without performing the actual blit/transfer to the display. Purpose unclear from the source; the implementation is empty. May have been intended for partial-screen updates or a double-buffering scheme.

**Parameters:** None.

**Return value:** None.

---

### `I_FinishUpdate`

**Signature:** `void I_FinishUpdate(void)`

**Purpose:** Transfers the completed framebuffer to the display. Called at the end of each frame after all rendering is complete. This is the function that actually causes the rendered image to become visible on screen.

**Parameters:** None.

**Return value:** None.

**Key logic:** On Linux/X11:
- If in developer mode, draws frame rate indicator dots on the bottom row.
- If display scaling is active (2x, 3x, 4x), performs pixel multiplication to expand the 320x200 buffer to the larger window size.
- Uses `XShmPutImage` (fast shared memory path) or `XPutImage` (standard path) to copy image data to the X window.
- Synchronizes with the X server.

---

### `I_WaitVBL`

**Signature:** `void I_WaitVBL(int count)`

**Purpose:** Waits for approximately `count` vertical blank intervals (each approximately 1/70th of a second). Used to introduce deliberate timing delays, such as the pause before the process exits after a quit confirmation, allowing a sound effect to play.

**Parameters:**
- `count` - Number of VBL intervals to wait (each ~14.3ms at 70Hz)

**Return value:** None.

**Key logic:** On Linux, implemented as `usleep(count * (1000000/70))`.

---

### `I_ReadScreen`

**Signature:** `void I_ReadScreen(byte* scr)`

**Purpose:** Copies the current contents of the primary framebuffer into the provided buffer. Used by the screenshot system (`M_ScreenShot`) to capture a frame for saving to a PCX file.

**Parameters:**
- `scr` - Destination buffer, must be at least `SCREENWIDTH * SCREENHEIGHT` (320 * 200 = 64000) bytes

**Return value:** None.

**Key logic:** `memcpy(scr, screens[0], SCREENWIDTH*SCREENHEIGHT)`.

---

### `I_BeginRead`

**Signature:** `void I_BeginRead(void)`

**Purpose:** Called before initiating a potentially slow disk read operation. Intended to display a disk activity indicator in the status bar area. No-op on Linux.

**Parameters:** None.

**Return value:** None.

**Key logic:** Empty on Linux. On DOS DOOM, this showed a floppy disk animation during WAD lump loading.

---

### `I_EndRead`

**Signature:** `void I_EndRead(void)`

**Purpose:** Called after a disk read completes. Clears the disk activity indicator shown by `I_BeginRead`. No-op on Linux.

**Parameters:** None.

**Return value:** None.

**Key logic:** Empty on Linux.

## Data Structures

No data structures are defined in this header. It uses:
- `byte` - DOOM's unsigned 8-bit integer type (from `doomtype.h`)

## Dependencies

| File | Reason |
|------|--------|
| `doomtype.h` | Defines `byte` (used as the parameter type for `I_SetPalette` and `I_ReadScreen`) |
