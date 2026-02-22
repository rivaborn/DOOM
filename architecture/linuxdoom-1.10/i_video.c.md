# File Overview

`i_video.c` implements DOOM's video output system for Linux/UNIX using the X11 windowing system. This is the platform-specific graphics backend that bridges DOOM's internal 320x200 8-bit framebuffer to an X11 display.

The file handles:
- **X11 window creation** and management
- **Input event processing** (keyboard, mouse, buttons) from X11 and translation to DOOM's event system
- **Screen blitting** with optional pixel multiplication (2x, 3x, or 4x scaling) for larger displays
- **Palette management** for 8-bit PseudoColor displays
- **MIT-SHM shared memory extension** for high-performance screen updates
- **Screen capture** for screenshots

The implementation is architecturally significant as it directly maps X11 events to DOOM's `event_t` type and uses the DOOM keyboard constants (e.g., `KEY_LEFTARROW`, `KEY_F1`) rather than raw X11 keysyms, providing complete input abstraction.

The blocky display multiplication feature (2x, 3x, 4x modes invoked with `-2`, `-3`, `-4` command-line flags) performs hand-optimized pixel expansion using 32-bit word manipulation to efficiently duplicate pixels.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `Display*` | `X_display` | Connection to the X11 display server. NULL until `I_InitGraphics` is called. |
| `Window` | `X_mainWindow` | The main X11 window handle where DOOM renders its output. |
| `Colormap` | `X_cmap` | The X11 colormap used for 8-bit PseudoColor display. Holds 256 color entries. |
| `Visual*` | `X_visual` | The X11 visual information pointer for the selected display mode. |
| `GC` | `X_gc` | The X11 graphics context used for drawing operations. |
| `XEvent` | `X_event` | Global storage for the current X11 event being processed. |
| `int` | `X_screen` | The X11 screen number being used (from `DefaultScreen`). |
| `XVisualInfo` | `X_visualinfo` | Detailed visual capabilities structure for the selected X11 visual. |
| `XImage*` | `image` | The XImage structure that wraps the pixel data buffer for transfer to X11. |
| `int` | `X_width` | The actual pixel width of the X11 window (SCREENWIDTH * multiply). |
| `int` | `X_height` | The actual pixel height of the X11 window (SCREENHEIGHT * multiply). |
| `boolean` | `doShm` | Flag indicating whether the MIT-SHM (shared memory) extension is being used. Faster than XPutImage when available. |
| `XShmSegmentInfo` | `X_shminfo` | Shared memory segment metadata used by the MIT-SHM extension. |
| `int` | `X_shmeventtype` | The X11 event type number for SHM completion events (used to synchronize with the X server). |
| `boolean` | `grabMouse` | Whether the mouse pointer is grabbed/confined to the DOOM window. Set by `-grabmouse` command line flag. |
| `int` | `doPointerWarp` | Countdown counter for mouse pointer warping back to window center. Prevents pointer from drifting out of window. |
| `static int` | `multiply` | Display scale factor: 1=320x200, 2=640x400, 3=960x600, 4=1280x800. Set by `-2`, `-3`, `-4` flags. |
| `static int` | `lastmousex` | Last recorded X position of the mouse pointer, for delta calculation. |
| `static int` | `lastmousey` | Last recorded Y position of the mouse pointer, for delta calculation. |
| `boolean` | `mousemoved` | Flag set when a mouse warp event should be ignored (to prevent feedback loops). |
| `boolean` | `shmFinished` | Flag set to true when the X server signals completion of a shared memory transfer. |
| `static XColor` | `colors[256]` | Array of 256 XColor structures used when uploading palette entries to the X11 colormap. |
| `unsigned` | `exptable[256]` | Lookup table for 4x pixel expansion: maps byte value to a 32-bit word with the byte replicated 4 times. Used by `InitExpand`. |
| `double` | `exptable2[256*256]` | Large lookup table for the `Expand4` function, maps two adjacent pixel values to a `double`-sized expanded form. |
| `int` | `inited` | Flag indicating whether `InitExpand2` has been called (lazy initialization guard for `Expand4`). |

## Functions

### `xlatekey`

**Signature:** `int xlatekey(void)`

**Purpose:** Translates the X11 keysym in the current `X_event` to DOOM's internal key code constants. This provides the input abstraction that lets DOOM's game logic use platform-independent key identifiers.

**Parameters:** None (reads from the global `X_event`).

**Return value:** Integer DOOM key code (one of the `KEY_*` constants from `doomdef.h`, or a lowercase ASCII character value).

**Key logic:**
- Calls `XKeycodeToKeysym` to convert the raw X11 keycode to a keysym.
- Uses a `switch` statement to map special X11 keysyms (arrow keys, function keys, modifiers, etc.) to DOOM constants.
- For printable ASCII characters (XK_space through XK_asciitilde), maps them to their ASCII values.
- Forces alphabetic keys to lowercase (DOOM uses lowercase for character comparisons).
- Both left and right variants of Shift/Ctrl/Alt are mapped to single DOOM constants (`KEY_RSHIFT`, `KEY_RCTRL`, `KEY_RALT`).

---

### `I_ShutdownGraphics`

**Signature:** `void I_ShutdownGraphics(void)`

**Purpose:** Releases the X11 shared memory segment and cleans up graphics resources. Called at program exit.

**Parameters:** None.

**Return value:** None.

**Key logic:**
1. Calls `XShmDetach` to detach the shared memory segment from the X server. Calls `I_Error` if this fails.
2. Calls `shmdt` to detach the shared memory from the DOOM process's address space.
3. Calls `shmctl(IPC_RMID)` to mark the shared memory segment for deletion by the kernel.
4. Sets `image->data = NULL` as a safety measure.

---

### `I_StartFrame`

**Signature:** `void I_StartFrame(void)`

**Purpose:** Called at the start of each display frame. No-op in this implementation.

**Parameters:** None.

**Return value:** None.

**Key logic:** Empty function body. The comment "er?" suggests this was intended for pre-frame setup but no Linux-specific action was needed.

---

### `I_GetEvent`

**Signature:** `void I_GetEvent(void)`

**Purpose:** Reads one pending X11 event and translates it into a DOOM `event_t`, posting it to the DOOM event queue via `D_PostEvent`.

**Parameters:** None (reads from `X_display`, writes to `X_event`).

**Return value:** None.

**Key logic:** Uses a `switch` on `X_event.type`:
- **KeyPress / KeyRelease**: Creates `ev_keydown` / `ev_keyup` events, translates the key with `xlatekey`.
- **ButtonPress**: Creates an `ev_mouse` event. `data1` encodes which mouse buttons are currently held (Button1=bit0, Button2=bit1, Button3=bit2) by OR-ing the current button state from `xbutton.state` with the newly pressed button from `xbutton.button`.
- **ButtonRelease**: Creates an `ev_mouse` event. XORs out the released button from the current state.
- **MotionNotify**: Creates an `ev_mouse` event with `data2` = horizontal delta (shifted left 2 bits), `data3` = vertical delta (shifted left 2 bits, inverted Y). Ignores warp events that return the pointer to center.
- **Expose / ConfigureNotify**: Ignored.
- **SHM completion event**: Sets `shmFinished = true` to signal that the shared memory transfer is complete.

---

### `createnullcursor`

**Signature:** `Cursor createnullcursor(Display* display, Window root)`

**Purpose:** Creates an invisible X11 cursor. Used to hide the mouse cursor within the DOOM window, providing a better game experience.

**Parameters:**
- `display` - The X11 display connection
- `root` - The root window (used as parent for cursor pixmap creation)

**Return value:** An X11 `Cursor` handle for the invisible cursor.

**Key logic:**
1. Creates a 1x1 pixel `Pixmap` for the cursor mask.
2. Creates a graphics context with `GXclear` function (everything drawn is cleared).
3. Fills the 1x1 pixmap with the clear GC (making it empty/transparent).
4. Creates a cursor from this empty pixmap using black color (making it invisible).
5. Frees the temporary pixmap and graphics context.

---

### `I_StartTic`

**Signature:** `void I_StartTic(void)`

**Purpose:** Called each game tic to collect pending input events from X11. This is the primary input polling point for the Linux port.

**Parameters:** None.

**Return value:** None.

**Key logic:**
1. Returns immediately if `X_display` is NULL (not initialized).
2. Calls `I_GetEvent()` in a loop while `XPending(X_display)` reports pending events.
3. If `grabMouse` is active, decrements `doPointerWarp` and when it reaches 0, warps the mouse cursor back to the window center (`X_width/2, X_height/2`) using `XWarpPointer`. This keeps the mouse confined to the window without a formal pointer grab. Resets `doPointerWarp` to `POINTER_WARP_COUNTDOWN` (1).
4. Resets `mousemoved = false` at the end.

---

### `I_UpdateNoBlit`

**Signature:** `void I_UpdateNoBlit(void)`

**Purpose:** Placeholder for a non-blitting screen update. Empty in this implementation.

**Parameters:** None.

**Return value:** None.

**Key logic:** Empty function body.

---

### `I_FinishUpdate`

**Signature:** `void I_FinishUpdate(void)`

**Purpose:** Transfers the contents of DOOM's internal framebuffer (`screens[0]`) to the X11 window. Handles optional pixel multiplication for larger display modes and performance indicator dots.

**Parameters:** None.

**Return value:** None.

**Key logic:**
1. **Developer mode dots**: If `devparm` is set, draws colored dots along the bottom row of `screens[0]` to indicate the current tic rate (performance indicator).
2. **Pixel multiplication (multiply == 2)**: Expands each input pixel to 2x2 pixels. Processes 4 input bytes at a time as a 32-bit integer, unpacks them, and writes each as 2 output bytes to 2 output scanlines using bitmask operations. Handles big-endian and little-endian byte order with `#ifdef __BIG_ENDIAN__`.
3. **Pixel multiplication (multiply == 3)**: Similar to 2x but expands to 3x3. Three output pointers advance in parallel.
4. **Pixel multiplication (multiply == 4)**: Calls `Expand4()` (noted as "Broken. Gotta fix this some day.").
5. **MIT-SHM path**: Calls `XShmPutImage` to transfer the image using shared memory. Then waits synchronously for the SHM completion event by calling `I_GetEvent` in a loop until `shmFinished` is true.
6. **Non-SHM path**: Calls `XPutImage` followed by `XSync` to ensure the image transfer completes.

---

### `I_ReadScreen`

**Signature:** `void I_ReadScreen(byte* scr)`

**Purpose:** Copies the current contents of `screens[0]` (DOOM's primary framebuffer) into the provided buffer. Used for taking screenshots.

**Parameters:**
- `scr` - Destination buffer, must be at least `SCREENWIDTH * SCREENHEIGHT` bytes

**Return value:** None.

**Key logic:** `memcpy(scr, screens[0], SCREENWIDTH*SCREENHEIGHT)` — a direct memory copy of the raw 320x200 framebuffer.

---

### `UploadNewPalette`

**Signature:** `void UploadNewPalette(Colormap cmap, byte *palette)`

**Purpose:** Uploads a new 256-color palette to the X11 colormap. Called when the game changes the color palette (e.g., red flash on damage, sector lighting changes).

**Parameters:**
- `cmap` - The X11 colormap to update
- `palette` - Pointer to 768 bytes of palette data (256 RGB triplets, each 0-255)

**Return value:** None.

**Key logic:**
1. Only operates if the visual is `PseudoColor` with 8-bit depth.
2. On the first call (`firstcall`), initializes the `colors` array with pixel indices and flags.
3. Iterates through all 256 palette entries. For each color, applies gamma correction by indexing into `gammatable[usegamma]`.
4. Converts 8-bit RGB values to 16-bit X11 color values: `(c<<8) + c` (maps 0-255 to 0-65535 with good distribution).
5. Calls `XStoreColors` to upload all 256 colors to the X11 colormap in a single call.

---

### `I_SetPalette`

**Signature:** `void I_SetPalette(byte* palette)`

**Purpose:** Public interface to update the display palette. Called by the game when the active PLAYPAL lump changes.

**Parameters:**
- `palette` - Pointer to 768 bytes of palette data (256 RGB triplets)

**Return value:** None.

**Key logic:** Delegates to `UploadNewPalette(X_cmap, palette)`.

---

### `grabsharedmemory`

**Signature:** `void grabsharedmemory(int size)`

**Purpose:** Acquires a System V shared memory segment for use with the MIT-SHM extension. Includes logic to reuse or clean up stale segments from previous DOOM instances (a workaround for processes that didn't clean up properly).

**Parameters:**
- `size` - Requested size of the shared memory segment in bytes

**Return value:** None. Sets `X_shminfo.shmid` and `image->data` / `X_shminfo.shmaddr` on success. Calls `I_Error` on failure.

**Key logic:**
1. Generates a deterministic key from the ASCII values of "doom" (`('d'<<24) | ('o'<<16) | ('o'<<8) | 'm'`).
2. Attempts to find an existing segment with that key. If found:
   - If currently attached (`shm_nattch > 0`): prints warning about another running DOOM, increments the key and retries.
   - If unattached and owned by current user: removes the old segment and creates a new one.
   - If unattached but large enough and owned by someone else: reuses the stale segment.
   - If not large enough: increments key and retries.
3. If no segment exists: creates one with `IPC_CREAT`.
4. Retries up to 5 times (`pollution` counter) before giving up with `I_Error`.
5. Attaches the segment with `shmat` and sets `image->data`.

---

### `I_InitGraphics`

**Signature:** `void I_InitGraphics(void)`

**Purpose:** Initializes the entire X11 graphics subsystem: opens a display connection, creates the window, sets up the colormap, graphics context, and XImage buffer (either shared memory or plain). Called once from `D_DoomMain`.

**Parameters:** None.

**Return value:** None.

**Key logic:** Protected by a `static int firsttime` flag to prevent re-initialization.
1. Installs `I_Quit` as the SIGINT handler.
2. Reads `-2`, `-3`, `-4` flags to set `multiply` (display scale factor).
3. Computes `X_width = SCREENWIDTH * multiply`, `X_height = SCREENHEIGHT * multiply`.
4. Reads `-disp` flag for explicit display name, else uses `DISPLAY` environment variable.
5. Reads `-grabmouse` flag to set `grabMouse`.
6. Reads `-geom` flag to parse window position (with sign prefix for negative coordinates).
7. Calls `XOpenDisplay` to connect to the X server.
8. Uses `XMatchVisualInfo` to find an 8-bit PseudoColor visual (exits with error if not found — DOOM requires 8-bit indexed color).
9. Tests for MIT-SHM availability with `XShmQueryExtension`. Disables SHM if the display is remote (not "unix" or empty hostname).
10. Creates the colormap with `XCreateColormap`.
11. Creates the main window with `XCreateWindow`, using the custom colormap and event mask (key press/release, expose).
12. Sets an invisible cursor with `createnullcursor`.
13. Creates the graphics context.
14. Maps the window and waits for the first Expose event before proceeding.
15. Optionally grabs the pointer with `XGrabPointer` if `grabMouse`.
16. **SHM path**: Calls `XShmGetEventBase`, creates the XImage with `XShmCreateImage`, calls `grabsharedmemory`, attaches with `XShmAttach`.
17. **Non-SHM path**: Creates XImage with `XCreateImage` backed by a `malloc`'d buffer.
18. Sets `screens[0]`: if `multiply==1`, points directly to `image->data`; otherwise allocates a separate 320x200 buffer.

---

### `InitExpand`

**Signature:** `void InitExpand(void)`

**Purpose:** Initializes `exptable`, a 256-entry lookup table for fast pixel doubling. Each entry maps a byte value `i` to the 32-bit integer `i | (i<<8) | (i<<16) | (i<<24)`, which replicates the byte in all four bytes of the word.

**Parameters:** None.

**Return value:** None.

---

### `InitExpand2`

**Signature:** `void InitExpand2(void)`

**Purpose:** Initializes `exptable2`, a 65536-entry lookup table used by `Expand4`. Each entry represents two adjacent input pixel values packed into a `double`, with each pixel's value replicated 4 times across 4 bytes.

**Parameters:** None.

**Return value:** None.

**Key logic:** Nested loop over all 256x256 combinations of byte values `i` and `j`. Uses a union of `double` and `unsigned[2]` to pack the expanded pixels: `u[0] = i|(i<<8)|(i<<16)|(i<<24)` and `u[1] = j|(j<<8)|(j<<16)|(j<<24)`. The resulting `double` is stored in `exptable2[i*256+j]`.

---

### `Expand4`

**Signature:** `void Expand4(unsigned* lineptr, double* xline)`

**Purpose:** Expands a 320x200 framebuffer to 1280x800 (4x scale) using `exptable2`. Noted in the source as broken.

**Parameters:**
- `lineptr` - Source 32-bit-aligned pointer into the 320x200 framebuffer
- `xline` - Destination `double`-aligned pointer into the output buffer

**Return value:** None.

**Key logic:** Lazily calls `InitExpand2` on first use. Iterates over all rows and columns, processing 16 input pixels (4 unsigned words) at a time. For each pair of input pixels (16 bits), looks up the pre-expanded value in `exptable2` and writes it to four output rows (0, 160, 320, 480 doubles ahead). The addressing uses pointer arithmetic and bit shifting to extract pixel pairs.

## Data Structures

No new data structures are defined in this file. It uses X11 types (`Display`, `Window`, `Colormap`, `Visual`, `GC`, `XEvent`, `XVisualInfo`, `XImage`, `XShmSegmentInfo`, `XColor`, `Cursor`) from the X11 headers.

## Dependencies

| File | Reason |
|------|--------|
| `<stdlib.h>` | `malloc` |
| `<unistd.h>` | UNIX standard functions |
| `<sys/ipc.h>`, `<sys/shm.h>` | System V shared memory (`shmget`, `shmat`, `shmdt`, `shmctl`) |
| `<X11/Xlib.h>` | Core X11 API |
| `<X11/Xutil.h>` | X11 utility functions |
| `<X11/keysym.h>` | X11 keysym constants (`XK_Left`, etc.) |
| `<X11/extensions/XShm.h>` | MIT-SHM extension API |
| `<stdarg.h>` | Variadic arguments (transitively needed) |
| `<sys/time.h>` | `gettimeofday` |
| `<sys/types.h>`, `<sys/socket.h>` | Socket types |
| `<netinet/in.h>` | Network byte order |
| `<signal.h>` | `signal()` for SIGINT handler |
| `doomstat.h` | Game state variables (`devparm`, `screens`, `gammatable`, `usegamma`) |
| `i_system.h` | `I_Error`, `I_Quit` |
| `v_video.h` | `screens` array (the framebuffer) |
| `m_argv.h` | `M_CheckParm`, `myargv` for command-line parsing |
| `d_main.h` | `D_PostEvent` for injecting events |
| `doomdef.h` | `SCREENWIDTH`, `SCREENHEIGHT`, `KEY_*` constants, `boolean`, `byte` |
