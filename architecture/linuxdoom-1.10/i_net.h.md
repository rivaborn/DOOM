# File Overview

`i_net.h` is the minimal public interface header for DOOM's platform-specific network system. It declares the two functions that the engine's portable networking code (`d_net.c`, `d_main.c`) calls to initialize and operate the network layer. The actual implementation is in `i_net.c`, which uses UDP sockets on Linux/Unix.

## Global Variables

None are declared in this header.

## Functions

### `I_InitNetwork`

```c
void I_InitNetwork(void);
```

Called by `D_DoomMain` during startup. Initializes the network subsystem: parses `-net` command-line arguments to determine whether this is a single-player or multiplayer session, resolves remote hostnames, creates UDP sockets, and sets up the `doomcom_t` structure used by the engine. In single-player mode, configures a minimal one-player `doomcom_t` without creating any sockets.

---

### `I_NetCmd`

```c
void I_NetCmd(void);
```

Called by the network tick code in `d_net.c` whenever a network operation is required. Dispatches to either the send or receive function (stored as function pointers in `i_net.c`) based on the command field in `doomcom`. Acts as the bridge between the engine's abstract network command interface and the platform-specific socket operations.

## Data Structures

None defined in this header.

## Dependencies

- This header has no explicit includes; callers are expected to have included the necessary type headers (`doomdef.h`, `d_net.h`) before including this file.
- GCC pragma `#pragma interface` is included for compatibility with g++ compilation.
