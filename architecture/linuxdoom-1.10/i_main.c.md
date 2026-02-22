# File Overview

`i_main.c` is the entry point for the DOOM Linux executable. It contains the standard C `main` function, which is the only function in this file. Its sole responsibility is to capture the command-line arguments into the global variables `myargc` and `myargv`, then transfer control to `D_DoomMain`, the high-level game initialization and main loop function. This thin wrapper keeps platform-specific entry-point code isolated from the engine's main logic.

## Global Variables

None are defined in this file. The global variables `myargc` and `myargv` are defined in `m_argv.c` and merely assigned here.

## Functions

### `main`

```c
int main(int argc, char** argv)
```

**Purpose:** Program entry point. Stores the command-line argument count and array into the engine's global argument variables, then starts the DOOM engine.

**Parameters:**
- `argc` — Number of command-line arguments (including the program name).
- `argv` — Array of C strings containing the command-line arguments.

**Returns:** `0` on normal exit. In practice, `D_DoomMain` never returns under normal operation — the game exits via `I_Quit` which calls `exit(0)` directly.

**Key logic:**
1. `myargc = argc` — Stores argument count globally for access throughout the engine.
2. `myargv = argv` — Stores argument array globally.
3. `D_DoomMain()` — Calls the engine's main initialization and game loop. This function does not return under normal circumstances.

## Data Structures

None.

## Dependencies

- `doomdef.h` — Core DOOM type definitions.
- `m_argv.h` — Declares `myargc` and `myargv` as externals (they are defined in `m_argv.c`).
- `d_main.h` — Declares `D_DoomMain`.
