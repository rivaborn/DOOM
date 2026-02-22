# File Overview

**File:** `ipx/IPXNET.H`
**Target Platform:** MS-DOS (16-bit real mode, Borland C)

`IPXNET.H` is the primary header for the IPX networking subsystem of the DOOM IPX driver. It defines all data structures needed to interface with the Novell IPX protocol: the Event Control Block (ECB) used for asynchronous I/O, the IPX packet header, address types, the composite packet buffer structure, and the setup handshake data structure. It also declares all global variables and function prototypes used between `IPXNET.C` and `IPXSETUP.C`.

The structures defined here directly map to the on-wire and in-memory layouts expected by the IPX protocol, so field byte ordering and alignment are critical.

---

## Global Variables

This is a header file. It declares the following external variables (defined in `IPXNET.C` and `IPXSETUP.C`):

| Type | Name | Defined In | Description |
|------|------|-----------|-------------|
| `extern doomcom_t` | `doomcom` | `IPXNETC`/`DOOMNET.C` | Shared DOOM communication block |
| `extern int` | `gameid` | `IPXSETUP.C` | Random game session ID used to allow multiple simultaneous DOOM sessions on the same network segment without interfering |
| `extern nodeadr_t` | `nodeadr[MAXNETNODES+1]` | `IPXNET.C` | Table of all participant IPX node addresses |
| `extern int` | `localnodenum` | `IPXSETUP.C` | This node's index in `nodeadr` |
| `extern long` | `localtime` | `IPXNET.C` | Logical packet send counter / timestamp |
| `extern long` | `remotetime` | `IPXNET.C` | Timestamp from most recently received packet |
| `extern nodeadr_t` | `remoteadr` | `IPXNET.C` | IPX node address of the sender of the last received packet |
| `extern int` | `myargc` | `IPXSETUP.C` | Argument count (mirrors `_argc`, possibly overridden by response file) |
| `extern char **` | `myargv` | `IPXSETUP.C` | Argument vector (mirrors `_argv`, possibly overridden by response file) |

---

## Functions

All function prototypes declared here:

### `Error`
```c
void Error(char *error, ...);
```
Variadic error handler. Prints a formatted message, shuts down the network, restores the interrupt vector if hooked, and exits. Defined in `IPXSETUP.C`.

### `InitNetwork`
```c
void InitNetwork(void);
```
Initializes the IPX interface, opens the DOOM socket, sets up receive/send ECBs, retrieves local address. Defined in `IPXNET.C`.

### `ShutdownNetwork`
```c
void ShutdownNetwork(void);
```
Closes the IPX socket. Defined in `IPXNET.C`.

### `SendPacket`
```c
void SendPacket(int destination);
```
Sends `doomcom.data` to the specified node. Defined in `IPXNET.C`.

### `GetPacket`
```c
int GetPacket(void);
```
Checks receive ECBs for an arrived packet; copies payload to `doomcom.data`. Returns 1 if a packet was received. Defined in `IPXNET.C`.

### `CheckParm`
```c
int CheckParm(char *check);
```
Searches `myargv` for a command-line parameter. Returns its index or 0. Defined in `IPXSETUP.C`.

### `PrintAddress`
```c
void PrintAddress(nodeadr_t *adr, char *str);
```
Formats a node address as a readable string. Utility function for diagnostics.

---

## Data Structures

### `doomdata_t`

```c
typedef struct
{
    char private[512];
} doomdata_t;
```

An opaque 512-byte buffer used as the DOOM packet payload within the `packet_t` structure. The name `private` (a C++ keyword, but valid in C) signals that the IPX layer treats this as an undifferentiated byte array — only DOOM interprets its contents.

---

### `setupdata_t`

```c
typedef struct
{
    short gameid;        // Unique session identifier to distinguish concurrent DOOM games
    short drone;         // 1 if this node is a drone (spectator), 0 if a player
    short nodesfound;    // How many nodes this machine has discovered so far
    short nodeswanted;   // Total number of nodes expected in this game
} setupdata_t;
```

Used in place of `doomdata_t` during the pre-game node discovery phase. Broadcast packets carry this structure so all machines can track when all expected players have been found. Layered over `doomcom.data` with a cast.

---

### Primitive Type Aliases

```c
typedef unsigned char  BYTE;
typedef unsigned short WORD;
typedef unsigned long  LONG;
```

Explicit-width type aliases matching the IPX protocol's documented field widths.

---

### `IPXPacket` (struct IPXPacketStructure)

```c
typedef struct IPXPacketStructure
{
    WORD PacketCheckSum;          // Packet checksum (usually 0xFFFF = not used)
    WORD PacketLength;            // Total packet length in bytes (big-endian)
    BYTE PacketTransportControl;  // Hop count (incremented by routers)
    BYTE PacketType;              // Packet type (4 = IPX)

    BYTE dNetwork[4];             // Destination network number (big-endian)
    BYTE dNode[6];                // Destination node address (MAC)
    BYTE dSocket[2];              // Destination socket number (big-endian)

    BYTE sNetwork[4];             // Source network number (big-endian)
    BYTE sNode[6];                // Source node address (MAC)
    BYTE sSocket[2];              // Source socket number (big-endian)
} IPXPacket;
```

The 30-byte IPX protocol header. All multi-byte fields are in big-endian (network) byte order, which requires byte swapping on x86. The driver fills `dNetwork`, `dNode`, and `dSocket` before sending; `sNetwork`, `sNode`, and `sSocket` are filled by the IPX driver hardware.

---

### `localadr_t`

```c
typedef struct
{
    BYTE network[4];   // 4-byte IPX network number (big-endian)
    BYTE node[6];      // 6-byte node address (usually the Ethernet MAC address)
} localadr_t;
```

Represents the complete IPX internetwork address of the local machine. Retrieved from the IPX driver via `GetLocalAddress()` during initialization.

---

### `nodeadr_t`

```c
typedef struct
{
    BYTE node[6];   // 6-byte IPX node address
} nodeadr_t;
```

Represents just the node portion of an IPX address. All players are assumed to be on the same IPX network segment (same network number), so only the node portion needs to be stored per player. The broadcast address is `{ 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF }`.

---

### `ECB` (struct ECBStructure)

```c
typedef struct ECBStructure
{
    WORD Link[2];              // Linked list pointers (maintained by IPX, not touched by app)
    WORD ESRAddress[2];        // Event Service Routine address (offset-segment); 0 = no ESR
    BYTE InUseFlag;            // Non-zero while IPX owns this ECB; zero when complete
    BYTE CompletionCode;       // Result code after IPX is done (0 = success)
    WORD ECBSocket;            // Socket number for this ECB (big-endian)
    BYTE IPXWorkspace[4];      // Reserved for IPX internal use
    BYTE DriverWorkspace[12];  // Reserved for network driver internal use
    BYTE ImmediateAddress[6];  // Physical (immediate) destination node address for routing
    WORD FragmentCount;        // Number of data fragments (little-endian)

    WORD fAddress[2];          // First fragment: offset-segment address
    WORD fSize;                // First fragment: byte count (little-endian)

    WORD f2Address[2];         // Second fragment: offset-segment address
    WORD f2Size;               // Second fragment: byte count (little-endian)
} ECB;
```

The Event Control Block is the fundamental structure for both sending and receiving with IPX. For receive ECBs (indices 1+), the fragment points to the `packet_t`'s IPX header and data. For the send ECB (index 0), two fragments are used: one for the IPX header+timestamp and one pointing directly into `doomcom.data`, allowing zero-copy sends.

**`InUseFlag`** is the key synchronization field: non-zero means IPX owns the ECB; zero means the application can read/process/reuse it.

---

### `packet_t`

```c
typedef struct
{
    ECB       ecb;       // IPX Event Control Block (must be first for IPX addressing)
    IPXPacket ipx;       // IPX protocol header
    long      time;      // Logical timestamp (set by sender, used for ordering)
    doomdata_t data;     // Game payload (512 bytes)
} packet_t;
```

The top-level packet buffer. The ECB is first in memory so that the ECB pointer equals the packet pointer. The `time` field is a 4-byte logical timestamp that DOOM uses to sequence packets when multiple arrive simultaneously. The `data` field holds the actual DOOM game data.

---

## Preprocessor Definitions

| Macro | Value | Description |
|-------|-------|-------------|
| `NUMPACKETS` | `10` | Total number of packet buffers in the pool. One is reserved for sending; the remaining 9 are posted as receive buffers. If all 9 fill up without being read, the oldest is dropped (packet loss). |

---

## Dependencies

| File | Reason |
|------|--------|
| `DoomNet.h` | Included internally to bring in `doomcom_t`, `MAXNETNODES`, `MAXPLAYERS`, `CMD_SEND`, `CMD_GET`, `DOOMCOM_ID`, and the `doomcom` extern |
