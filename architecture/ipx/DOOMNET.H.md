# File Overview

**File:** `ipx/DOOMNET.H`
**Target Platform:** MS-DOS (16-bit real mode)

`DOOMNET.H` is the central header shared between the IPX network driver and the DOOM game executable. It defines the hardware-level palette I/O macros, the fundamental game-network constants, and most importantly the `doomcom_t` structure — the shared memory communication block that allows DOOM and its external network driver to exchange commands and packet data at runtime.

This header establishes the protocol contract: DOOM allocates or locates a `doomcom_t` block via the `-net` command-line argument (which provides the block's flat address), then uses a CPU software interrupt to signal the driver process whenever it needs to send or receive a network packet.

---

## Global Variables

This is a header file. It declares the following external variables (defined in `DOOMNET.C`):

| Type | Name | Description |
|------|------|-------------|
| `extern doomcom_t` | `doomcom` | The shared communication block. One instance lives in the driver's data segment and its flat address is passed to DOOM on the command line. |
| `extern void interrupt (*olddoomvect)(void)` | `olddoomvect` | The original interrupt handler that was in place before `NetISR` was installed. Restored on exit. |
| `extern int` | `vectorishooked` | Non-zero when the interrupt vector has been hooked and needs to be restored on exit. |

---

## Functions

The following function prototypes are declared here (implemented elsewhere):

### `CheckParm`
```c
int CheckParm(char *check);
```
Searches `myargv` for the given string (case-insensitive). Returns the index of the argument (1-based) or 0 if not found.

### `LaunchDOOM`
```c
void LaunchDOOM(void);
```
Hooks the interrupt vector and spawns the DOOM executable as a child process with the `-net` argument pointing to the `doomcom` block. See `DOOMNET.C` for full documentation.

### `NetISR`
```c
void interrupt NetISR(void);
```
The interrupt service routine installed by `LaunchDOOM`. Called by the DOOM engine when it needs to send (`doomcom.command == CMD_SEND`) or receive (`doomcom.command == CMD_GET`) a network packet. The `interrupt` keyword is a Borland C extension that generates a full interrupt stack frame (saves/restores all registers and uses `iret` to return).

---

## Data Structures

### `doomcom_t`

```c
typedef struct
{
    long  id;              // Magic number: must equal DOOMCOM_ID (0x12345678)
    short intnum;          // The software interrupt number used for driver communication

    // Communication fields (set by DOOM before firing the interrupt)
    short command;         // CMD_SEND (1) or CMD_GET (2)
    short remotenode;      // For CMD_SEND: destination node index; for CMD_GET: set by driver to source node (-1 = no packet)
    short datalength;      // For CMD_SEND: number of bytes in data[]; for CMD_GET: set by driver to received byte count

    // Game configuration (set by driver before launching DOOM)
    short numnodes;        // Total number of network nodes (console is always node 0)
    short ticdup;          // Tic duplication factor: 1 = no duplication, 2-5 = send extra copies for slow links
    short extratics;       // Extra tic backup count: 1 = send one previous tic in each packet
    short deathmatch;      // 0 = cooperative, 1 = deathmatch
    short savegame;        // -1 = new game, 0-5 = load specified savegame slot
    short episode;         // Starting episode (1-3 for DOOM, unused for DOOM II)
    short map;             // Starting map (1-9)
    short skill;           // Skill level (1-5)

    // Node-specific configuration
    short consoleplayer;   // This node's player number (0-3)
    short numplayers;      // Total number of human players (1-4, excludes drones)
    short angleoffset;     // View angle offset: 1 = left, 0 = center, -1 = right
    short drone;           // 1 if this node is a spectator/drone, 0 otherwise

    // Packet payload
    char  data[512];       // Raw packet bytes to send (CMD_SEND) or received bytes (CMD_GET)
} doomcom_t;
```

**Design Notes:**
- The structure lives in the driver's data segment. Its flat (linear) memory address is passed to DOOM via the `-net` command-line argument as a decimal integer. DOOM reconstructs a far pointer from this address.
- `id` is the first field; DOOM verifies it equals `DOOMCOM_ID` to confirm the structure is valid.
- `intnum` is chosen at startup (typically 0x60-0x66) as a free interrupt vector. `LaunchDOOM` installs `NetISR` at that vector and DOOM calls it using the CPU `INT` instruction.
- `data[512]` is the raw payload area used for both outgoing and incoming packets.

---

## Preprocessor Definitions

| Macro | Value | Description |
|-------|-------|-------------|
| `PEL_WRITE_ADR` | `0x3c8` | VGA palette write address port |
| `PEL_DATA` | `0x3c9` | VGA palette data port |
| `I_ColorBlack(r,g,b)` | (inline port writes) | Debug macro to set the screen border/palette entry 0 to a given RGB color — used in commented-out debug code to visually indicate interrupt activity |
| `MAXNETNODES` | `8` | Maximum number of computers that can participate in a network game |
| `MAXPLAYERS` | `4` | Maximum number of human players (drones not counted) |
| `CMD_SEND` | `1` | Command code placed in `doomcom.command` by DOOM when it wants to send a packet |
| `CMD_GET` | `2` | Command code placed in `doomcom.command` by DOOM when it wants to receive a packet |
| `DOOMCOM_ID` | `0x12345678L` | Magic identification value written into `doomcom.id` before launching DOOM |

---

## Dependencies

| File | Reason |
|------|--------|
| None (self-contained header) | This header does not include other project headers |

Standard library headers are included by the `.c` files that include this header, not by the header itself.
