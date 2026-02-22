# File Overview

`DOOMNET.H` is the shared header file for the DOOM DOS network driver subsystem. It is included by both the serial link driver (`SERSETUP.C`, `PORT.C`, `DOOMNET.C`) and the IPX network driver. It defines the central `doomcom_t` structure, which is the shared memory interface through which the DOOM game engine communicates with whichever external network driver is active.

The architecture it supports works as follows: before launching DOOM, a network driver (serial or IPX) sets up a `doomcom_t` structure in its own memory, hooks a software interrupt vector, and passes the flat (real-mode) memory address of the structure to DOOM via the `-net <address>` command-line argument. Whenever DOOM needs to send or receive a packet, it writes a command into `doomcom.command` and fires the interrupt. The driver's ISR reads from or writes to `doomcom.data` accordingly.

This header also provides macros for VGA palette manipulation (`I_ColorBlack`) that were used for debugging — setting the screen border color to indicate ISR activity. These are typically commented out in the ISR code but left defined here.

---

## Global Variables

The following are declared `extern` in this header, meaning they are defined in `DOOMNET.C` and shared across all compilation units that include this header:

| Type | Name | Description |
|------|------|-------------|
| `doomcom_t` | `doomcom` | The shared communication block between DOOM and the network driver. |
| `void interrupt (*olddoomvect)(void)` | `olddoomvect` | Saved pointer to the original interrupt vector, for restoration on exit. |
| `int` | `vectorishooked` | Flag indicating whether the interrupt vector has been hooked. |

---

## Functions

The following function prototypes are declared in this header:

### `CheckParm`

**Signature:**
```c
int CheckParm(char *check);
```

Defined in `DOOMNET.C`. Searches command-line arguments for a given string (case-insensitive). Returns the argument index if found, or 0 if not found.

---

### `LaunchDOOM`

**Signature:**
```c
void LaunchDOOM(void);
```

Defined in `DOOMNET.C`. Hooks a software interrupt vector, builds the argument list with the `-net <address>` parameters, and spawns the DOOM executable as a child process.

---

### `NetISR`

**Signature:**
```c
void interrupt NetISR(void);
```

Defined in the driver's main source file (e.g., `SERSETUP.C` for serial, or the IPX driver's equivalent). This is the interrupt service routine that DOOM calls to send or receive network packets. It inspects `doomcom.command` and performs the appropriate I/O operation.

---

## Data Structures

### `doomcom_t`

The central inter-process communication structure between DOOM and the network driver.

```c
typedef struct
{
    long    id;             // Magic number: must be DOOMCOM_ID (0x12345678)
    short   intnum;         // Software interrupt number DOOM uses to communicate

    // Command interface (written by DOOM before firing the interrupt)
    short   command;        // CMD_SEND (1) or CMD_GET (2)
    short   remotenode;     // Destination node for send; set by driver on get (-1 = no packet)
    short   datalength;     // Number of bytes in data[] to send, or bytes received

    // Game configuration fields (written by the driver before launching DOOM)
    short   numnodes;       // Total number of nodes; node 0 is always the console
    short   ticdup;         // Tic duplication factor: 1 = none, 2-5 = for slow networks
    short   extratics;      // 1 = send a backup tic in every packet for reliability
    short   deathmatch;     // 1 = deathmatch mode
    short   savegame;       // -1 = new game, 0-5 = load from save slot
    short   episode;        // Episode number (1-3 for DOOM 1)
    short   map;            // Map number (1-9)
    short   skill;          // Skill level (1-5)

    // Per-node configuration (specific to this machine's node)
    short   consoleplayer;  // This player's number (0-3)
    short   numplayers;     // Total number of human players (1-4)
    short   angleoffset;    // View angle offset: 1=left, 0=center, -1=right
    short   drone;          // 1 if this node is a drone (spectator, not a player)

    // Packet payload
    char    data[512];      // Raw packet data to be sent or received
} doomcom_t;
```

**Field Details:**

- **`id`**: DOOM validates this field equals `DOOMCOM_ID` (`0x12345678L`) before using the structure. Acts as a sanity check against a bad memory address.
- **`intnum`**: The software interrupt number (typically `0x60`-`0x66`) that DOOM will use to trigger the driver's `NetISR`. Set by the driver in `LaunchDOOM`.
- **`command`**: Set by DOOM before triggering the interrupt. `CMD_SEND` (1) means DOOM wants to transmit the data in `data[]`; `CMD_GET` (2) means DOOM wants to receive any pending packet.
- **`remotenode`**: On `CMD_SEND`, DOOM sets this to the destination node index. On `CMD_GET`, the driver sets this to the source node index of the received packet, or `-1` if no packet is available.
- **`datalength`**: On `CMD_SEND`, DOOM sets this to the number of bytes to transmit. On `CMD_GET`, the driver sets this to the number of bytes received.
- **`numnodes`** through **`skill`**: Configuration filled in by the driver before calling `LaunchDOOM`, based on command-line arguments or negotiation with the remote machine.
- **`data[512]`**: The actual packet payload buffer, shared between DOOM and the driver via the interrupt interface.

---

## Preprocessor Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `PEL_WRITE_ADR` | `0x3c8` | VGA DAC palette write address register (used for debug color flashing) |
| `PEL_DATA` | `0x3c9` | VGA DAC palette data register |
| `MAXNETNODES` | `8` | Maximum number of nodes (computers) in a network game |
| `MAXPLAYERS` | `4` | Maximum number of human players |
| `CMD_SEND` | `1` | Command value: DOOM wants to send a packet |
| `CMD_GET` | `2` | Command value: DOOM wants to receive a packet |
| `DOOMCOM_ID` | `0x12345678L` | Magic identifier that DOOM checks to validate the doomcom structure |

### `I_ColorBlack(r,g,b)` Macro

```c
#define I_ColorBlack(r,g,b) { outp(PEL_WRITE_ADR,0); outp(PEL_DATA,r); outp(PEL_DATA,g); outp(PEL_DATA,b); }
```

Sets VGA palette color index 0 (the border/overscan color) to the specified RGB values. Used as a visual debugging tool in ISRs — for example, setting the border red during a transmit interrupt and green during a receive interrupt. All actual calls to this macro in the driver code are commented out.

---

## Dependencies

| File | Reason |
|------|--------|
| None (system headers) | This is a pure header file with no `#include` directives of its own |

This header is included by:
- `DOOMNET.C` (defines the variables declared `extern` here)
- `SERSETUP.C` (uses `doomcom`, `vectorishooked`, `olddoomvect`, `CMD_SEND`, `CMD_GET`)
- `PORT.C` (uses `doomcom` indirectly via the ISR context)
- The IPX driver (`IPXNET.C`, `IPXSETUP.C`) — same header is reused across both driver types
