# File Overview

`i_system.c` is the platform-specific system services implementation for the Linux/UNIX port of DOOM. It implements the functions declared in `i_system.h`, providing the operating system interface between the DOOM engine and the underlying UNIX platform.

This file is part of the "I_" (Implementation/Interface) layer, which isolates the rest of the engine from OS-specific details. Functions in this layer handle time measurement, memory allocation, system initialization/shutdown, error handling, and timing delays.

Notable characteristics of this file:
- Uses `gettimeofday()` for high-precision time measurement converted to 1/70th-second tics (DOOM's native timing unit)
- Provides 6 MB of heap memory by default for the zone memory system
- Implements a clean shutdown sequence that saves settings, closes the network game, and shuts down audio and video
- Contains stubs for platform-specific features (e.g., `I_Tactile` is empty, `I_BeginRead`/`I_EndRead` are no-ops on Linux)

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `int` | `mb_used` | The amount of heap memory (in megabytes) allocated for DOOM's zone memory system. Defaults to 6 MB. May be overridden via the `mb_used` config file entry under `SNDSERV` mode. |
| `ticcmd_t` | `emptycmd` | A zero-initialized `ticcmd_t` structure returned by `I_BaseTiccmd`. Represents a "null" input command with no buttons pressed and no movement. |
| `boolean` | `demorecording` | External variable (defined elsewhere, declared `extern` here). Used by `I_Error` to check if a demo is being recorded; if so, it calls `G_CheckDemoStatus` to close the demo file before terminating. |

## Functions

### `I_Tactile`

**Signature:** `void I_Tactile(int on, int off, int total)`

**Purpose:** Stub for tactile (force feedback / rumble) device control. Originally intended for joystick force feedback. Not implemented on Linux.

**Parameters:**
- `on` - Duration of the "on" pulse in some unit (unused)
- `off` - Duration of the "off" interval (unused)
- `total` - Total duration (unused)

**Return value:** None.

**Key logic:** Immediately zeros out all parameters and returns. Effectively a no-op. The comment reads "UNUSED."

---

### `I_BaseTiccmd`

**Signature:** `ticcmd_t* I_BaseTiccmd(void)`

**Purpose:** Returns a pointer to an empty (all-zeros) tic command structure. This serves as the starting point for building each frame's input command. The game loop then modifies this base command with actual player input.

**Parameters:** None.

**Return value:** Pointer to the global `emptycmd` structure (always zero-initialized, representing no input).

**Key logic:** Returns the address of the static `emptycmd` global. No logic — this is a simple accessor. On DOS DOOM, this could return a command pre-loaded from a joystick driver; on Linux it is always empty.

---

### `I_GetHeapSize`

**Signature:** `int I_GetHeapSize(void)`

**Purpose:** Returns the size of the heap (in bytes) that DOOM's zone memory manager should allocate.

**Parameters:** None.

**Return value:** `mb_used * 1024 * 1024` — the heap size in bytes (defaults to 6,291,456 bytes / 6 MB).

**Key logic:** Simple multiplication of `mb_used` by 1 MB. The value of `mb_used` can be changed by the config file system (`M_LoadDefaults`) when running with `SNDSERV` support.

---

### `I_ZoneBase`

**Signature:** `byte* I_ZoneBase(int* size)`

**Purpose:** Allocates and returns the base pointer for DOOM's zone memory heap. Called once at startup by the zone memory initializer (`Z_Init`).

**Parameters:**
- `size` - Output parameter: set to the number of bytes allocated (same as `I_GetHeapSize()` return value)

**Return value:** Pointer to a freshly `malloc`'d block of `mb_used * 1024 * 1024` bytes. Ownership transfers to the zone memory system.

**Key logic:** Sets `*size`, calls `malloc(*size)`, and returns the result. If `malloc` fails, the returned pointer is NULL and the zone system will crash — there is no error handling here.

---

### `I_GetTime`

**Signature:** `int I_GetTime(void)`

**Purpose:** Returns the current game time in 1/70th-second tics (DOOM's standard time unit, matching the original PC timer frequency of 70 Hz).

**Parameters:** None.

**Return value:** Integer tic count since the first call to this function, measured in units of 1/70th of a second.

**Key logic:**
1. Calls `gettimeofday()` to get the current wall-clock time as seconds + microseconds.
2. On the first call, records `basetime` (the seconds component at startup) to prevent integer overflow.
3. Computes elapsed time: `(tv_sec - basetime) * TICRATE + tv_usec * TICRATE / 1000000`.
4. `TICRATE` is 35 (defined in `doomdef.h`), but the comment says 1/70th second — this is because the formula gives 35 tics/second, which at the nominal 35 tic/second rate matches the original game. (The comment "1/70th second tics" is slightly misleading; TICRATE=35 means 35 tics per second.)

Uses a `static int basetime` to store the epoch, initialized to zero and set on first call.

---

### `I_Init`

**Signature:** `void I_Init(void)`

**Purpose:** Performs system-level initialization at engine startup. Called by `D_DoomMain` during the early startup phase.

**Parameters:** None.

**Return value:** None.

**Key logic:** Calls `I_InitSound()` to start the audio system. The call to `I_InitGraphics()` is commented out (graphics are initialized later in `D_DoomMain` via a direct call).

---

### `I_Quit`

**Signature:** `void I_Quit(void)`

**Purpose:** Performs a clean, orderly shutdown of DOOM and exits the process normally. Called when the user chooses "Quit" from the menu.

**Parameters:** None.

**Return value:** Does not return (calls `exit(0)`).

**Key logic:** Executes the full shutdown sequence in order:
1. `D_QuitNetGame()` - Sends quit notification to other network players
2. `I_ShutdownSound()` - Stops and releases sound effects system
3. `I_ShutdownMusic()` - Stops and releases music system
4. `M_SaveDefaults()` - Writes current settings to the config file (e.g., `default.cfg`)
5. `I_ShutdownGraphics()` - Closes the X11 window and releases graphics resources
6. `exit(0)` - Terminates the process with success code

---

### `I_WaitVBL`

**Signature:** `void I_WaitVBL(int count)`

**Purpose:** Waits for approximately `count` vertical blanking intervals. Used to introduce timing delays (e.g., before quitting, to let a quit sound play).

**Parameters:**
- `count` - Number of VBL intervals to wait (approximately 1/70th second each)

**Return value:** None.

**Key logic:** Platform-conditional:
- On SGI: calls `sginap(1)` (not a real wait, essentially yields)
- On SUN: calls `sleep(0)` (effectively no-op)
- On Linux/default: calls `usleep(count * (1000000/70))` — sleeps for `count * 14285` microseconds, which is `count` frames at 70 Hz

---

### `I_BeginRead`

**Signature:** `void I_BeginRead(void)`

**Purpose:** Called before a slow disk read operation to show a disk activity indicator. No-op on Linux.

**Parameters:** None.

**Return value:** None.

**Key logic:** Empty function body. On DOS DOOM this would show a floppy disk icon in the status bar during WAD loading.

---

### `I_EndRead`

**Signature:** `void I_EndRead(void)`

**Purpose:** Called after a slow disk read operation to hide the disk activity indicator. No-op on Linux.

**Parameters:** None.

**Return value:** None.

**Key logic:** Empty function body. Counterpart to `I_BeginRead`.

---

### `I_AllocLow`

**Signature:** `byte* I_AllocLow(int length)`

**Purpose:** Allocates a zero-initialized block of memory. Under DOS, this allocated from "low memory" (below 640K) for DMA-accessible buffers. Under UNIX/Linux, it simply calls `malloc` and `memset`.

**Parameters:**
- `length` - Number of bytes to allocate

**Return value:** Pointer to a freshly allocated and zeroed block of `length` bytes.

**Key logic:** Calls `malloc(length)` then `memset(mem, 0, length)` to zero-initialize the block. No error handling if `malloc` returns NULL.

---

### `I_Error`

**Signature:** `void I_Error(char *error, ...)`

**Purpose:** Handles fatal errors. Prints an error message to stderr, performs emergency shutdown, and exits the process with a failure code. This is DOOM's equivalent of a fatal assertion failure.

**Parameters:**
- `error` - printf-style format string for the error message
- `...` - Variadic arguments for the format string

**Return value:** Does not return (calls `exit(-1)`).

**Key logic:**
1. Uses `va_start`/`vfprintf`/`va_end` to print the formatted error message to `stderr` with an "Error: " prefix and trailing newline.
2. Flushes `stderr`.
3. If `demorecording` is true, calls `G_CheckDemoStatus()` to close the demo file cleanly.
4. Calls `D_QuitNetGame()` to notify other players of the crash.
5. Calls `I_ShutdownGraphics()` to close the X11 window (avoids leaving orphaned windows).
6. Calls `exit(-1)` to terminate with an error code.

Note: Unlike `I_Quit`, this does NOT save defaults or shut down sound, as those operations might themselves fail.

## Data Structures

No new data structures are defined in this file. Uses `ticcmd_t` from `d_ticcmd.h`.

## Dependencies

| File | Reason |
|------|--------|
| `<stdlib.h>` | `malloc`, `exit` |
| `<stdio.h>` | `fprintf`, `vfprintf`, `fflush` |
| `<string.h>` | `memset` |
| `<stdarg.h>` | Variadic argument macros (`va_list`, `va_start`, `va_end`) |
| `<sys/time.h>` | `gettimeofday`, `struct timeval`, `struct timezone` |
| `<unistd.h>` | `usleep` |
| `doomdef.h` | `TICRATE`, `byte` type, and other fundamental definitions |
| `m_misc.h` | `M_SaveDefaults` (called in `I_Quit`) |
| `i_video.h` | `I_ShutdownGraphics` (called in `I_Quit` and `I_Error`) |
| `i_sound.h` | `I_InitSound`, `I_ShutdownSound`, `I_ShutdownMusic` |
| `d_net.h` | `D_QuitNetGame` |
| `g_game.h` | `G_CheckDemoStatus` |
| `i_system.h` | Own header (self-implementation) |
