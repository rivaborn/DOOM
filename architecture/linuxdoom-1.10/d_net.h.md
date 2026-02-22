# File Overview

`d_net.h` is the public header for DOOM's network communication layer. It defines the two key data structures used for all multiplayer communication — the packet format (`doomdata_t`) and the shared communication block (`doomcom_t`) — along with constants controlling packet sizes and the command interface. It also declares the three public functions of the network subsystem that the rest of the engine calls.

Every module that participates in the network game (directly or indirectly) includes this header, since `doomstat.h` includes it to expose the `doomcom` and `netbuffer` extern pointers.

## Global Variables

This header declares no global variables. The `doomcom` and `netbuffer` pointers declared as `extern` in `doomstat.h` are defined in `d_net.c`.

## Functions

### `NetUpdate`

```c
void NetUpdate(void);
```

**Purpose:** Creates any new tic commands for the console player and broadcasts them to all other networked nodes. Also receives and processes any incoming packets.

**Called from:** `TryRunTics` (inside `d_net.c`) and the main loop in `d_main.c`.

---

### `D_QuitNetGame`

```c
void D_QuitNetGame(void);
```

**Purpose:** Broadcasts exit notification packets to all other players before the game terminates. Called during clean shutdown.

---

### `TryRunTics`

```c
void TryRunTics(void);
```

**Purpose:** Determines how many game tics to run for the current frame, waits for network data if necessary, and executes those tics. This is the main game-loop driver when networking is active.

## Data Structures

### `command_t` (enum)

The command code placed in `doomcom_t::command` to tell the network driver what to do:

| Value | Name | Meaning |
|-------|------|---------|
| `1` | `CMD_SEND` | Transmit the packet in `doomcom->data` to `doomcom->remotenode`. |
| `2` | `CMD_GET` | Receive the next available packet into `doomcom->data`; sets `remotenode` to the sender or -1 if none. |

---

### `doomdata_t` (struct)

The actual payload transmitted over the network. Contains tic commands for one or more game ticks.

```c
typedef struct {
    unsigned  checksum;        // Packet flags (top 4 bits) and checksum (bottom 28 bits).
    byte      retransmitfrom;  // If NCMD_RETRANSMIT: the tic to retransmit from.
    byte      starttic;        // Low byte of the first tic number in cmds[].
    byte      player;          // Sending player number (or VERSION during setup).
    byte      numtics;         // Number of ticcmd_t entries in cmds[].
    ticcmd_t  cmds[BACKUPTICS];// The tic command data.
} doomdata_t;
```

Field details:
- **`checksum`**: The top 4 bits encode control flags (`NCMD_EXIT`, `NCMD_RETRANSMIT`, `NCMD_SETUP`, `NCMD_KILL`). The bottom 28 bits are the actual data checksum computed by `NetbufferChecksum()`.
- **`retransmitfrom`**: During `NCMD_SETUP` packets, repurposed to carry game option bits (skill, deathmatch, nomonsters, respawn flags).
- **`starttic`**: Only the low 8 bits of the actual tic number are sent; the receiver calls `ExpandTics()` to reconstruct the full value.
- **`cmds[BACKUPTICS]`**: The variable-length tail; only `numtics` entries are actually transmitted in any given packet.

---

### `doomcom_t` (struct)

The shared memory block between the DOOM executable and the network driver. This structure is filled by `I_InitNetwork()` and subsequently used to exchange commands and data between the game and the OS/driver layer.

```c
typedef struct {
    long    id;              // Must be DOOMCOM_ID (0x12345678) for validation.
    short   intnum;          // Software interrupt number used by the driver.
    short   command;         // CMD_SEND or CMD_GET (set by DOOM, read by driver).
    short   remotenode;      // Destination (send) or source (get); -1 = no packet.
    short   datalength;      // Byte count of valid data in the 'data' field.
    short   numnodes;        // Total number of network nodes (including self).
    short   ticdup;          // Tic duplication factor (1 = no dup, 2-5 = slower nets).
    short   extratics;       // If 1, an extra tic is included as backup in each packet.
    short   deathmatch;      // 1 = deathmatch mode.
    short   savegame;        // -1 = new game, 0-5 = load from savegame slot.
    short   episode;         // Starting episode (1-3).
    short   map;             // Starting map (1-9).
    short   skill;           // Skill level (1-5).
    short   consoleplayer;   // This node's player number (0 = key player).
    short   numplayers;      // Number of human players.
    short   angleoffset;     // For 3-screen display: 1=left, 0=center, -1=right.
    short   drone;           // 1 if this node is a drone (view-only).
    doomdata_t data;         // The network packet buffer.
} doomcom_t;
```

Key design notes:
- `consoleplayer == 0` designates the "key player" who controls game start parameters in `D_ArbitrateNetStart`.
- `ticdup` allows slow network links by duplicating game tics: a value of 2 means every network tic covers 2 game tics, halving bandwidth requirements.
- The `drone` and `angleoffset` fields supported an experimental 3-monitor mode (two drone computers rendering left/right views) that was likely not operational by the time the source was released.
- `data` is placed last so that `doomdata_t` can be used as a variable-length structure; only `datalength` bytes of it are transmitted.

## Dependencies

| File | Reason |
|------|--------|
| `d_player.h` | Required for `ticcmd_t` (used inside `doomdata_t::cmds`) and for `MAXPLAYERS`. |

### Constants Defined

| Constant | Value | Meaning |
|----------|-------|---------|
| `DOOMCOM_ID` | `0x12345678L` | Magic identifier verifying the `doomcom_t` block is valid. |
| `MAXNETNODES` | `8` | Maximum number of computers in a networked game. |
| `BACKUPTICS` | `12` | Size of the circular tic command buffers; limits how far behind a node can fall. |
