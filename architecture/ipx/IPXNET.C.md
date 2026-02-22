# File Overview

**File:** `ipx/IPXNET.C`
**Target Platform:** MS-DOS (16-bit real mode, Borland C)

`IPXNET.C` implements the low-level IPX (Internetwork Packet Exchange) networking layer for the DOOM IPX multiplayer driver. It is responsible for all direct communication with the Novell NetWare IPX protocol stack through the IPX API entry point obtained from interrupt 0x2F, and provides the packet send/receive functions that `IPXSETUP.C` and `NetISR` call at a higher level.

IPX was the dominant LAN protocol for DOS-era office and home networks (typically running over Novell NetWare or compatible IPX stacks). This file communicates with the IPX driver via a far function pointer obtained through a multiplex interrupt call, then builds and manages Event Control Blocks (ECBs) to queue receive buffers and send packets.

The module maintains a pool of `packet_t` structures: index 0 is always the send ECB, and indices 1 through `NUMPACKETS-1` are pre-queued receive ECBs (posted to IPX so the driver can fill them asynchronously).

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `packet_t packets[NUMPACKETS]` | `packets` | Pool of packet buffers. `packets[0]` is the sending ECB; `packets[1..NUMPACKETS-1]` are listening receive ECBs posted to IPX. Each entry contains an ECB, an IPX header, a timestamp, and a data buffer. |
| `nodeadr_t nodeadr[MAXNETNODES+1]` | `nodeadr` | Table of 6-byte IPX node addresses for all participants. Index 0 is the local machine, index `MAXNETNODES` (8) is the broadcast address (all 0xFF bytes). Filled in during the setup handshake. |
| `nodeadr_t remoteadr` | `remoteadr` | Holds the source node address extracted from the most recently received packet. Updated each time `GetPacket()` successfully reads a packet. |
| `localadr_t localadr` | `localadr` | The local machine's full IPX address (4-byte network number + 6-byte node number). Filled in by `GetLocalAddress()` during `InitNetwork()`. |
| `void far (*IPX)(void)` | `IPX` | Far function pointer to the IPX API entry point, obtained via interrupt 0x2F multiplex call `AX=0x7A00`. All IPX operations are invoked by loading registers and calling through this pointer. |
| `long localtime` | `localtime` | Logical timestamp incremented by `NetISR` each time a packet is sent. Set to `-1` during the setup/discovery phase, then to `0` when gameplay begins. Used to sequence received packets and to distinguish setup packets from game packets. |
| `long remotetime` | `remotetime` | Timestamp extracted from the most recently received packet. Used by `GetPacket()` to detect and discard out-of-phase setup traffic. |
| `extern int socketid` | `socketid` | The IPX socket number used for all DOOM network traffic (defined in `IPXSETUP.C`; the official DOOM socket is `0x869C`). |

---

## Functions

### `OpenSocket`

**Signature:**
```c
int OpenSocket(short socketNumber)
```

**Purpose:**
Opens an IPX socket for network communication by calling the IPX API with function code 0 (Open Socket).

**Parameters:**
- `socketNumber` — The IPX socket number to open. Note that IPX sockets are stored in big-endian (high-low) byte order, so the caller must swap bytes before passing here when using a well-known little-endian socket number.

**Return Value:** The opened socket number as returned by the IPX driver (in `_DX`). Calls `Error()` and terminates if IPX returns a non-zero error code in `_AL`.

**Key Logic:**
Sets `_DX = socketNumber`, `_BX = 0` (function: Open Socket), `_AL = 0` (long-lived socket), then calls through the `IPX` function pointer. Reads error code from `_AL` and socket from `_DX`.

---

### `CloseSocket`

**Signature:**
```c
void CloseSocket(short socketNumber)
```

**Purpose:**
Closes a previously opened IPX socket by calling IPX function code 1 (Close Socket). Called by `ShutdownNetwork()` on exit.

**Parameters:**
- `socketNumber` — The socket number to close.

**Return Value:** `void`.

---

### `ListenForPacket`

**Signature:**
```c
void ListenForPacket(ECB *ecb)
```

**Purpose:**
Posts an ECB to the IPX driver as a listening receive buffer (IPX function code 4: Listen for Packet). IPX will fill this ECB asynchronously when a packet arrives on the registered socket. The `ecb->InUseFlag` will be non-zero while IPX holds the buffer, and zero when a packet has been received.

**Parameters:**
- `ecb` — Pointer to an Event Control Block that has been pre-configured with socket, fragment address, and fragment size.

**Return Value:** `void`. Calls `Error()` if IPX reports an error in `_AL`.

---

### `GetLocalAddress`

**Signature:**
```c
void GetLocalAddress(void)
```

**Purpose:**
Retrieves the local machine's IPX internetwork address (4-byte network + 6-byte node) by calling IPX function code 9 (Get Internetwork Address). Stores the result in the global `localadr` structure.

**Parameters:** None.

**Return Value:** `void`.

---

### `InitNetwork`

**Signature:**
```c
void InitNetwork(void)
```

**Purpose:**
Performs all one-time initialization of the IPX networking layer:
1. Locates the IPX API entry point.
2. Opens the DOOM socket.
3. Retrieves the local IPX address.
4. Pre-posts `NUMPACKETS-1` receive ECBs to IPX.
5. Configures the single send ECB.
6. Populates `nodeadr[0]` with the local node and `nodeadr[MAXNETNODES]` with the broadcast address.

**Parameters:** None.

**Return Value:** `void`. Calls `Error()` and terminates if IPX is not detected.

**Key Logic:**

1. **Detect IPX:** Issues `INT 0x2F` with `AX=0x7A00`. If `AL != 0xFF`, IPX is not installed. Otherwise, `ES:DI` contains the far pointer to the IPX API entry point, which is stored in `IPX`.

2. **Open socket:** `socketid` from `IPXSETUP.C` is byte-swapped (IPX socket numbers are big-endian) before being passed to `OpenSocket()`.

3. **Set up receive ECBs:** For each receive slot (indices 1..`NUMPACKETS-1`), zeroes the buffer, sets `ECBSocket`, sets `FragmentCount = 1`, sets the fragment address (`fAddress`) to the IPX header portion of the `packet_t`, sets `fSize` to the total non-ECB size of `packet_t`, and calls `ListenForPacket()`.

4. **Set up send ECB:** Zeroes `packets[0]`, sets `ECBSocket`, sets `FragmentCount = 2` (one fragment for the IPX header + timestamp, one for `doomcom.data`), fills the destination network number from `localadr.network`, and sets the socket fields. The second fragment (`f2Address`) points directly at `doomcom.data`, so the driver reads outgoing payload directly from the shared DOOM communication block without copying.

5. **Address table:** Copies local node bytes to `nodeadr[0]`; fills `nodeadr[MAXNETNODES]` with `0xFF` bytes (broadcast).

---

### `ShutdownNetwork`

**Signature:**
```c
void ShutdownNetwork(void)
```

**Purpose:**
Closes the IPX socket if the `IPX` pointer is non-null (i.e., if `InitNetwork()` succeeded). Called on exit to release the socket.

**Parameters:** None.

**Return Value:** `void`.

---

### `SendPacket`

**Signature:**
```c
void SendPacket(int destination)
```

**Purpose:**
Sends the data currently in `doomcom.data` (of length `doomcom.datalength`) to the node at index `destination` in the `nodeadr` table. A destination of `MAXNETNODES` is the broadcast address.

**Parameters:**
- `destination` — Index into `nodeadr[]` identifying the target node. `MAXNETNODES` (8) = broadcast.

**Return Value:** `void`. Calls `Error()` on IPX send error.

**Key Logic:**
1. Sets `packets[0].time = localtime` (the logical timestamp).
2. Copies the destination node's 6-byte address into both `packets[0].ipx.dNode` (IPX header destination node) and `packets[0].ecb.ImmediateAddress` (the physical next-hop address for IPX routing).
3. Sets `fSize` to `sizeof(IPXPacket) + 4` (IPX header plus 4-byte time field) and `f2Size` to `doomcom.datalength + 4` (game payload plus 4 bytes).
4. Calls IPX function 3 (Send Packet) via the `IPX` function pointer.
5. **Polls for completion:** Spins in a loop calling IPX function 10 (Relinquish Control) while `packets[0].ecb.InUseFlag != 0`. This is essential for polled IPX drivers (as opposed to interrupt-driven ones) to actually process the send.

---

### `ShortSwap`

**Signature:**
```c
unsigned short ShortSwap(unsigned short i)
```

**Purpose:**
Swaps the two bytes of a 16-bit value, converting between little-endian (x86) and big-endian (IPX network) byte order.

**Parameters:**
- `i` — The 16-bit value to byte-swap.

**Return Value:** The byte-swapped value.

---

### `GetPacket`

**Signature:**
```c
int GetPacket(void)
```

**Purpose:**
Checks all receive ECBs for a completed (arrived) packet. If multiple packets are waiting, returns the one with the oldest (smallest) `time` value to preserve ordering. Copies the packet payload into `doomcom.data`, sets `doomcom.datalength`, identifies the source node by matching the sender address against `nodeadr[]`, and re-posts the ECB to IPX for the next receive.

**Parameters:** None.

**Return Value:**
- `1` — A valid packet was received and placed into `doomcom.data`; `doomcom.remotenode` identifies the source.
- `0` — No packet is ready, or the packet was discarded (setup packet during game, or unknown sender).

**Key Logic:**
1. **Find oldest packet:** Iterates receive ECBs (indices 1..`NUMPACKETS-1`). Skips any where `ecb.InUseFlag` is non-zero (still held by IPX). Tracks the entry with the minimum `time` value.

2. **No packet:** If `besttic == MAXLONG`, no ECB has completed; returns 0.

3. **Setup packet filter:** If `besttic == -1` (timestamp used during setup phase) and `localtime != -1` (game has started), this is a stale setup broadcast from another instance. Re-posts the ECB and returns 0.

4. **Node identification:** Copies `packet->ipx.sNode` into `remoteadr`, then searches `nodeadr[0..numnodes-1]` for a match. If found, sets `doomcom.remotenode` to that index. If not found and we are in game mode (`localtime != -1`), discards the packet and re-posts.

5. **Data extraction:** Computes `datalength` as `ShortSwap(packet->ipx.PacketLength) - 38`. The 38 subtracted is the size of the fixed IPX header (30 bytes) plus 4 bytes for the time field plus 4 bytes for... the IPX overhead. Copies data from `packet->data` into `doomcom.data`.

6. **Re-post ECB:** Calls `ListenForPacket()` on the just-consumed ECB to return it to the IPX driver.

---

## Data Structures

This file uses structures defined in `ipxnet.h`:

- **`ECB`** — Event Control Block used by IPX to manage asynchronous send and receive operations.
- **`IPXPacket`** — The 30-byte IPX packet header (checksum, length, transport control, type, destination/source network+node+socket).
- **`packet_t`** — Composite structure containing an ECB, an IPX header, a `long` timestamp, and a `doomdata_t` payload buffer.
- **`nodeadr_t`** — 6-byte IPX node address.
- **`localadr_t`** — 10-byte full IPX internetwork address (4-byte network + 6-byte node).

---

## Dependencies

| File | Reason |
|------|--------|
| `<stdio.h>` | `printf` (via `Error()`) |
| `<stdlib.h>` | General utilities |
| `<dos.h>` | Register variables (`_AX`, `_BX`, `_DX`, `_ES`, `_DI`, `_SI`), `geninterrupt`, `FP_OFF`, `FP_SEG`, `MK_FP` |
| `<string.h>` | `memset`, `memcpy`, `memcmp` |
| `<process.h>` | Spawn utilities |
| `<values.h>` | `MAXLONG` constant |
| `ipxnet.h` | All data structure definitions (`ECB`, `IPXPacket`, `packet_t`, `nodeadr_t`, `localadr_t`, `setupdata_t`, `NUMPACKETS`, `MAXNETNODES`) and extern declarations |
| `doomnet.h` | (included via `ipxnet.h`) `doomcom_t`, `doomcom`, `CMD_SEND`, `CMD_GET`, `MAXNETNODES` |
