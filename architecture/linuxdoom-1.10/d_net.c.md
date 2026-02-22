# File Overview

`d_net.c` implements the core OS-independent network communication and protocol layer for DOOM multiplayer games. It is responsible for building and transmitting tic commands to other players, receiving and processing incoming packets, synchronizing the game loop across all networked nodes, and managing the startup handshake between players. It sits above the platform-specific network driver (`i_net.c`) and below the game logic (`g_game.c`).

The fundamental design is a lockstep simulation: every node must have received tic data from all other nodes before it can advance the game state. The module buffers recent tic commands, detects missed packets and requests retransmission, and adapts timing to keep all peers in sync.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `doomcom_t*` | `doomcom` | Pointer to the shared communication structure used by both DOOM and the network driver. Set by `I_InitNetwork`. |
| `doomdata_t*` | `netbuffer` | Points into `doomcom->data`; the working buffer for composing and reading network packets. |
| `ticcmd_t[BACKUPTICS]` | `localcmds` | Circular buffer storing the local console player's tic commands waiting to be sent. |
| `ticcmd_t[MAXPLAYERS][BACKUPTICS]` | `netcmds` | Circular buffer of received tic commands for all players. Indexed by player number and tic. |
| `int[MAXNETNODES]` | `nettics` | The highest tic number received from each network node (the `maketic` of that node). |
| `boolean[MAXNETNODES]` | `nodeingame` | True for each node that is still connected and playing. |
| `boolean[MAXNETNODES]` | `remoteresend` | Set true when a node needs tics we haven't sent yet (i.e., we need to request retransmission from remote). |
| `int[MAXNETNODES]` | `resendto` | The tic number to start resending from for each node. |
| `int[MAXNETNODES]` | `resendcount` | Countdown used to throttle retransmit requests per node. |
| `int[MAXPLAYERS]` | `nodeforplayer` | Maps player number to network node number. |
| `int` | `maketic` | The next tic number for which the local player's command has not yet been built. |
| `int` | `lastnettic` | Tracks the last network tic (used for timing adaptation). |
| `int` | `skiptics` | Number of tics to skip when the local node is running too fast. |
| `int` | `ticdup` | Tic duplication factor from `doomcom`; controls how many real tics each network tic represents. |
| `int` | `maxsend` | Maximum number of tic commands to include in a single packet: `BACKUPTICS/(2*ticdup)-1`. |
| `boolean` | `reboundpacket` | True when a loopback packet is waiting in `reboundstore` (used for single-node testing). |
| `doomdata_t` | `reboundstore` | Stores the most recently "sent" packet for loopback (node 0 sends to itself). |
| `char[80]` | `exitmsg` | Buffer for the "Player X left the game" message displayed when a remote node exits. |
| `int` | `gametime` | Tracks the last known game time (in tic units divided by `ticdup`) for detecting new tics. |
| `int[4]` | `frametics` | Per-frame tic counts used for timing adaptation. |
| `int` | `frameon` | Frame counter, used to index into `frameskip`. |
| `int[4]` | `frameskip` | Circular buffer of booleans recording whether recent frames were skipped; used to detect when the local node is consistently slow. |
| `int` | `oldnettics` | Previous value of the key player's `nettics`, used for adaptation. |

## Functions

### `NetbufferSize`

```c
int NetbufferSize(void)
```

**Purpose:** Calculates the actual byte size of the current `netbuffer` packet, which varies depending on how many tic commands (`numtics`) it contains.

**Parameters:** None.

**Returns:** `int` — byte count from the start of `doomdata_t` through the last `ticcmd_t` in the `cmds` array.

**Key logic:** Uses pointer arithmetic on a null-cast `doomdata_t*` to compute the offset of `cmds[netbuffer->numtics]`, giving the total packet size.

---

### `NetbufferChecksum`

```c
unsigned NetbufferChecksum(void)
```

**Purpose:** Computes a 28-bit checksum over the data portion of `netbuffer` (from `retransmitfrom` onward) to detect corrupted or mismatched packets.

**Parameters:** None.

**Returns:** `unsigned` — checksum value masked to 28 bits (`& NCMD_CHECKSUM`). Returns 0 unconditionally under `NORMALUNIX` due to byte-order issues.

**Key logic:** Seeds with `0x1234567`, sums 32-bit words weighted by position. The result is ANDed with `NCMD_CHECKSUM` (0x0fffffff) so the top 4 bits remain available for command flags.

---

### `ExpandTics`

```c
int ExpandTics(int low)
```

**Purpose:** Reconstructs the full 32-bit tic count from the low 8-bit value transmitted in packets, using the current `maketic` as a reference.

**Parameters:**
- `low` — The 8-bit tic value from the packet.

**Returns:** `int` — The full tic number, correctly accounting for byte-wraparound.

**Key logic:** Computes the delta between `low` and the low byte of `maketic`. If within ±64, no wraparound has occurred. If `delta > 64`, the packet is from a previous 256-tic window; if `delta < -64`, from the next window.

---

### `HSendPacket`

```c
void HSendPacket(int node, int flags)
```

**Purpose:** Sets the packet checksum and flags, then dispatches `netbuffer` to a specific network node. Handles the special case of node 0 (loopback).

**Parameters:**
- `node` — The destination node index (0 = local loopback).
- `flags` — Bitfield of `NCMD_*` flags OR'd into the checksum field.

**Returns:** void.

**Key logic:**
1. Stores `NetbufferChecksum() | flags` in `netbuffer->checksum`.
2. If `node == 0`, copies `netbuffer` to `reboundstore` and sets `reboundpacket = true` (simulates network send to self).
3. If demo playback, returns early (no real network needed).
4. Sets `doomcom->command = CMD_SEND`, `doomcom->remotenode = node`, and `doomcom->datalength = NetbufferSize()`, then calls `I_NetCmd()`.
5. Optionally logs the packet to `debugfile`.

---

### `HGetPacket`

```c
boolean HGetPacket(void)
```

**Purpose:** Attempts to receive one network packet. Returns false if no packet is available.

**Parameters:** None.

**Returns:** `boolean` — true if a valid packet was received into `netbuffer`, false otherwise.

**Key logic:**
1. If `reboundpacket` is set, restores from `reboundstore` (loopback delivery).
2. Otherwise sets `doomcom->command = CMD_GET` and calls `I_NetCmd()`.
3. Returns false if `doomcom->remotenode == -1` (no packet).
4. Validates the received packet's length and checksum; logs and discards on mismatch.

---

### `GetPackets`

```c
void GetPackets(void)
```

**Purpose:** Drains all available incoming packets, updating `netcmds` with received tic commands and responding to control messages (exit, kill, retransmit requests).

**Parameters:** None.

**Returns:** void.

**Key logic:**
- Loops calling `HGetPacket()` until no more packets remain.
- Skips setup packets (already handled by `D_ArbitrateNetStart`).
- Handles `NCMD_EXIT`: removes the sending player from the game and displays the exit message.
- Handles `NCMD_KILL`: calls `I_Error` (network driver forcibly ends the game).
- Handles `NCMD_RETRANSMIT`: records the requested resend starting tic in `resendto`.
- Detects out-of-order or duplicate packets by comparing `realend` against `nettics[netnode]`.
- Detects missed packets by checking if `realstart > nettics[netnode]`, setting `remoteresend[netnode] = true` to trigger a retransmit request.
- Copies received `ticcmd_t` entries into the `netcmds` circular buffer.

---

### `NetUpdate`

```c
void NetUpdate(void)
```

**Purpose:** The main network tick: builds new local tic commands for any new real-time ticks, broadcasts them to all other nodes, and receives incoming packets.

**Parameters:** None.

**Returns:** void.

**Key logic:**
1. Reads the current time and calculates `newtics` (how many new tic slots to fill).
2. Applies `skiptics` correction (if local node is running too fast).
3. For each new tic, calls `I_StartTic`, `D_ProcessEvents`, and `G_BuildTiccmd` to produce a `ticcmd_t` stored in `localcmds`.
4. If not in single-tic mode, sends `netbuffer` to every active node, starting from `resendto[i]` (which may be an earlier tic if retransmission was requested).
5. Calls `GetPackets()` to process any received data.

---

### `CheckAbort`

```c
void CheckAbort(void)
```

**Purpose:** Polls for an Escape keypress during the network startup handshake, aborting with an error if detected. Also pumps the event queue.

**Parameters:** None.

**Returns:** void.

**Key logic:** Busy-waits for ~2 real-time ticks via `I_GetTime`, then scans the event queue. If any `ev_keydown` event has `KEY_ESCAPE`, calls `I_Error` to abort.

---

### `D_ArbitrateNetStart`

```c
void D_ArbitrateNetStart(void)
```

**Purpose:** Performs the network game startup handshake. The key player (node 0) broadcasts game parameters; other players listen and extract those parameters.

**Parameters:** None.

**Returns:** void.

**Key logic:**
- Sets `autostart = true`.
- If `doomcom->consoleplayer != 0` (non-key player): waits for a `NCMD_SETUP` packet and extracts `startskill`, `deathmatch`, `nomonsters`, `respawnparm`, `startmap`, and `startepisode` from the packet's fields.
- If key player: repeatedly sends `NCMD_SETUP` packets encoding all game parameters to all other nodes, collecting acknowledgments until all nodes have replied.
- Calls `CheckAbort()` in each iteration to allow the user to cancel.

---

### `D_CheckNetGame`

```c
void D_CheckNetGame(void)
```

**Purpose:** Initializes all networking state, calls `I_InitNetwork`, validates the `doomcom` structure, runs the startup arbitration, and sets up game-side parameters from `doomcom`.

**Parameters:** None.

**Returns:** void.

**Key logic:**
1. Resets all `nodeingame`, `nettics`, `remoteresend`, and `resendto` arrays.
2. Calls `I_InitNetwork()` which fills in `doomcom` and sets `netgame`.
3. Validates `doomcom->id == DOOMCOM_ID`.
4. Sets `netbuffer = &doomcom->data`.
5. Sets `consoleplayer` and `displayplayer` from `doomcom->consoleplayer`.
6. If `netgame`, calls `D_ArbitrateNetStart`.
7. Reads `ticdup` and `maxsend` from `doomcom`.
8. Marks `playeringame` and `nodeingame` arrays based on `doomcom->numplayers` and `doomcom->numnodes`.

---

### `D_QuitNetGame`

```c
void D_QuitNetGame(void)
```

**Purpose:** Notifies other network nodes that this player is leaving, sends multiple redundant exit packets to ensure delivery, and closes the debug file.

**Parameters:** None.

**Returns:** void.

**Key logic:** Sends four rounds of `NCMD_EXIT` packets to all in-game nodes (with `I_WaitVBL(1)` between rounds for spacing), then returns. Does nothing if not in a netgame, or if in demo playback.

---

### `TryRunTics`

```c
void TryRunTics(void)
```

**Purpose:** The main game loop driver. Determines how many tic updates to run this frame based on real time and network availability, waits for all peers to supply enough tic data, then executes the appropriate number of game tics.

**Parameters:** None.

**Returns:** void.

**Key logic:**
1. Computes `realtics` (real time elapsed since last call) and `availabletics` (tics available from the slowest network node).
2. Calls `NetUpdate()` to refresh network state.
3. Finds the minimum `nettics` across all active nodes (`lowtic`).
4. Decides `counts` = min(realtics, availabletics) with various adjustments.
5. Runs an adaptive timing correction: if the local node is consistently behind the key player, decrements `gametime` and sets `skiptics` to speed up.
6. Busy-waits (calling `NetUpdate()` repeatedly) until `lowtic` catches up with what's needed, giving the menu a chance to run every ~20 tics.
7. Runs `counts * ticdup` tic updates by calling `D_DoAdvanceDemo`, `M_Ticker`, `G_Ticker`, and incrementing `gametic`. For duplicated tics, clears chat and special button data to prevent them from being processed multiple times.

## Data Structures

No new structs are defined in this file. It uses `doomcom_t`, `doomdata_t`, and `ticcmd_t` from `d_net.h`.

### Packet Command Flags (local `#define`s)

| Constant | Value | Meaning |
|----------|-------|---------|
| `NCMD_EXIT` | `0x80000000` | The sending node is leaving the game. |
| `NCMD_RETRANSMIT` | `0x40000000` | The packet requests retransmission from a given tic. |
| `NCMD_SETUP` | `0x20000000` | A startup handshake/setup packet. |
| `NCMD_KILL` | `0x10000000` | The network driver is forcibly ending the game. |
| `NCMD_CHECKSUM` | `0x0fffffff` | Mask for the checksum portion of the `checksum` field. |

Additional local constants:
- `RESENDCOUNT 10` — Cooldown ticks between honoring retransmit requests.
- `PL_DRONE 0x80` — Bit flag in `doomdata->player` indicating a drone (viewing-only) node.

## Dependencies

| File | Reason |
|------|--------|
| `m_menu.h` | `M_Ticker()` called during busy-wait; `M_StartControlPanel` indirectly. |
| `i_system.h` | `I_Error`, `I_GetTime`, `I_WaitVBL`, `I_StartTic`, `I_BaseTiccmd`. |
| `i_video.h` | `I_ReadScreen` (indirectly through the broader video system). |
| `i_net.h` | `I_InitNetwork`, `I_NetCmd` — platform-specific network driver calls. |
| `g_game.h` | `G_BuildTiccmd`, `G_Ticker`, `G_CheckDemoStatus`. |
| `doomdef.h` | `BACKUPTICS`, `MAXPLAYERS`, `boolean`, `VERSION`, key constants. |
| `doomstat.h` | Global game state: `netgame`, `demoplayback`, `demorecording`, `players`, `playeringame`, `consoleplayer`, `gametic`, `debugfile`, `singletics`, etc. |
