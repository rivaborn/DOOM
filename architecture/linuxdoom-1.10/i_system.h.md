# File Overview

`i_system.h` is the platform-abstraction header for DOOM's system services layer. It declares the interface that the portable game engine uses to interact with operating-system-specific services such as memory allocation, time measurement, error handling, and input.

This header defines the contract between the game engine (the "D_" and "G_" modules) and the platform-specific implementation layer (the "I_" modules). By adhering to this interface, the DOOM codebase can be ported to new platforms by replacing the "I_" implementation files while leaving all game logic untouched.

The `#pragma interface` directive (when compiled with g++) enables GNU C++ interface/implementation separation optimizations.

## Global Variables

No global variables are declared in this header. All system state is managed within the implementation file `i_system.c`.

## Functions

### `I_Init`

**Signature:** `void I_Init(void)`

**Purpose:** Performs all platform-specific initialization at engine startup. Called once by `D_DoomMain` before the main game loop begins.

**Parameters:** None.

**Return value:** None.

**Key logic:** Initializes sound and any other platform hardware. On Linux, this calls `I_InitSound()`. Graphics initialization is handled separately via a direct call to `I_InitGraphics` later in the startup sequence.

---

### `I_ZoneBase`

**Signature:** `byte* I_ZoneBase(int *size)`

**Purpose:** Allocates and returns the base memory block for DOOM's zone memory heap. Called by the zone memory system (`Z_Init`) during startup to obtain the raw memory that will be managed as a heap.

**Parameters:**
- `size` - Output parameter: set to the number of bytes in the returned allocation. The caller uses this to initialize the zone manager's bookkeeping.

**Return value:** Pointer to a freshly allocated block of memory. The zone memory system takes ownership of this block.

**Key logic:** Implementation-defined. On Linux, `malloc`s a block of `mb_used * 1024 * 1024` bytes (default 6 MB) and sets `*size` to that value.

---

### `I_GetTime`

**Signature:** `int I_GetTime(void)`

**Purpose:** Returns the current game time in tics (1/35th second units on the Linux implementation, corresponding to DOOM's internal 35 Hz tick rate). Called continuously by `D_DoomLoop` to drive the game simulation.

**Parameters:** None.

**Return value:** Elapsed time in tics since the engine started, as an integer. Monotonically increasing.

**Key logic:** Uses the system real-time clock (`gettimeofday` on Linux) to compute the elapsed time and converts it to tics. The formula uses `TICRATE` (35) as the conversion factor.

---

### `I_StartFrame`

**Signature:** `void I_StartFrame(void)`

**Purpose:** Called by `D_DoomLoop` at the beginning of each display frame, before any tics for that frame are processed. Intended for time-consuming synchronous operations (such as reading a joystick) that should happen once per frame, not once per tic.

**Parameters:** None.

**Return value:** None.

**Key logic:** Can call `D_PostEvent` to inject events into the event queue. On Linux, the X11 input is read in `I_StartTic` instead, so this may be a no-op.

---

### `I_StartTic`

**Signature:** `void I_StartTic(void)`

**Purpose:** Called by `D_DoomLoop` before processing each game tic. Reads pending input events and posts them to the event queue via `D_PostEvent`. This is where keyboard, mouse, and joystick events are collected.

**Parameters:** None.

**Return value:** None.

**Key logic:** On Linux, calls `XPending` in a loop to drain the X11 event queue, translating X events to DOOM `event_t` structures and posting them. Also handles the mouse warping back to the window center if mouse grabbing is active.

---

### `I_BaseTiccmd`

**Signature:** `ticcmd_t* I_BaseTiccmd(void)`

**Purpose:** Returns a pointer to the base tic command for the current tic. The game loop uses this as the starting point for building the player's input command, then modifies it based on event processing.

**Parameters:** None.

**Return value:** Pointer to an empty (all-zeros) `ticcmd_t` structure. On DOS, this could return a command pre-populated by a loadable input driver. On Linux, it returns a zero structure.

**Key logic:** On Linux, always returns a pointer to the static `emptycmd` global (an all-zero `ticcmd_t`).

---

### `I_Quit`

**Signature:** `void I_Quit(void)`

**Purpose:** Performs an orderly shutdown of the entire DOOM engine. Called when the user confirms the "Quit DOOM" prompt. Responsible for saving the config file, disconnecting from network games, and shutting down all subsystems before exiting.

**Parameters:** None.

**Return value:** Does not return (terminates the process via `exit(0)`).

**Key logic:** Calls `D_QuitNetGame`, `I_ShutdownSound`, `I_ShutdownMusic`, `M_SaveDefaults`, `I_ShutdownGraphics`, then `exit(0)`.

---

### `I_AllocLow`

**Signature:** `byte* I_AllocLow(int length)`

**Purpose:** Allocates a zero-initialized block of memory. The name reflects the DOS origin of this function, where it specifically allocated from DOS "low memory" (below 640K) for DMA buffers. On UNIX/Linux, it is equivalent to a `calloc`.

**Parameters:**
- `length` - Number of bytes to allocate

**Return value:** Pointer to a freshly allocated, zero-initialized block of `length` bytes.

**Key logic:** On Linux, implemented as `malloc(length)` followed by `memset(mem, 0, length)`.

---

### `I_Tactile`

**Signature:** `void I_Tactile(int on, int off, int total)`

**Purpose:** Controls a tactile feedback (force feedback / rumble) device. Intended for joystick force feedback on supported hardware.

**Parameters:**
- `on` - Duration of the feedback pulse
- `off` - Duration of the off interval
- `total` - Total duration of the feedback sequence

**Return value:** None.

**Key logic:** Not implemented on Linux (stub function that does nothing).

---

### `I_Error`

**Signature:** `void I_Error(char *error, ...)`

**Purpose:** Handles a fatal error condition. Prints a formatted error message, performs emergency shutdown, and exits the process with a failure code. Used throughout the engine wherever an unrecoverable error is detected.

**Parameters:**
- `error` - printf-style format string for the error message
- `...` - Variadic format arguments

**Return value:** Does not return (exits the process via `exit(-1)`).

**Key logic:** Prints the message to `stderr`, calls `G_CheckDemoStatus` if a demo is being recorded, calls `D_QuitNetGame` and `I_ShutdownGraphics`, then exits with error code -1.

## Data Structures

No data structures are defined in this header. It uses the following types from other headers:

- `ticcmd_t` from `d_ticcmd.h` — the input command structure passed to the game tic processor
- `byte` from DOOM's type definitions — an unsigned 8-bit integer

## Dependencies

| File | Reason |
|------|--------|
| `d_ticcmd.h` | Defines `ticcmd_t`, used as the return type of `I_BaseTiccmd` |
| `d_event.h` | Defines `event_t` and `D_PostEvent`, referenced conceptually (implementations call `D_PostEvent`) |
