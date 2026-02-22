# File Overview

`i_net.c` is the Linux/Unix platform-specific implementation of DOOM's network layer. It provides UDP socket-based peer-to-peer multiplayer networking, translating the engine's abstract `doomcom_t` network commands into actual Berkeley socket operations. When `-net` is specified on the command line, this module creates UDP sockets, resolves hostnames, and dispatches packets between players. For single-player games, it sets up a minimal `doomcom_t` structure indicating one local player. The module uses function pointers (`netsend`, `netget`) so that the core networking code (`d_net.c`) can call send and receive operations without knowing the transport details.

**Note:** This file contains a custom byte-swap macro definition that overrides the standard `ntohl`/`ntohs`/`htonl`/`htons` macros from `<netinet/in.h>`. The comment says "For some odd reason..."

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `int` | `DOOMPORT` | UDP port number used for multiplayer. Defaults to `IPPORT_USERRESERVED + 0x1d` (5029). Can be overridden with `-port` command-line parameter. |
| `int` | `sendsocket` | File descriptor for the UDP socket used to send packets. |
| `int` | `insocket` | File descriptor for the UDP socket used to receive packets. Set to non-blocking mode. |
| `struct sockaddr_in` | `sendaddress[MAXNETNODES]` | Array of destination addresses, one per remote player node. Populated during `I_InitNetwork` from the `-net` argument's hostname list. |
| `void (*)(void)` | `netget` | Function pointer to the receive function. Set to `PacketGet` for networked games, NULL for single-player. |
| `void (*)(void)` | `netsend` | Function pointer to the send function. Set to `PacketSend` for networked games, NULL for single-player. |

### Byte-Swap Macros (local overrides)

The file locally redefines the standard network byte-order macros:

```c
#define ntohl(x)  /* big-endian to little-endian 32-bit */
#define ntohs(x)  /* big-endian to little-endian 16-bit */
#define htonl(x)  ntohl(x)
#define htons(x)  ntohs(x)
```

These are defined inline rather than using the system versions.

## Functions

### `UDPsocket`

```c
int UDPsocket(void)
```

**Purpose:** Creates and returns a new UDP datagram socket.

**Parameters:** None.
**Returns:** The socket file descriptor.

**Key logic:** Calls `socket(PF_INET, SOCK_DGRAM, IPPROTO_UDP)`. On failure, calls `I_Error`.

---

### `BindToLocalPort`

```c
void BindToLocalPort(int s, int port)
```

**Purpose:** Binds a socket to the local machine's address on the specified port, accepting datagrams from any interface (`INADDR_ANY`).

**Parameters:**
- `s` — Socket file descriptor.
- `port` — Port number (in network byte order).

**Returns:** Nothing. Calls `I_Error` on failure.

---

### `PacketSend`

```c
void PacketSend(void)
```

**Purpose:** Sends the contents of `netbuffer` (the engine's shared network data buffer) to the remote node specified by `doomcom->remotenode`, with appropriate byte-swapping for network transmission.

**Parameters:** None (operates on global `netbuffer` and `doomcom`).
**Returns:** Nothing.

**Key logic:**
- Copies `netbuffer` fields into a local `doomdata_t` structure `sw`, converting multi-byte fields to network byte order using `htonl` and `htons`.
- Fields converted: `checksum` (32-bit), `angleturn` and `consistancy` in each `ticcmd_t` (16-bit each).
- Single-byte fields (`player`, `retransmitfrom`, `starttic`, `numtics`, `forwardmove`, `sidemove`, `chatchar`, `buttons`) are copied directly.
- Calls `sendto` to transmit to `sendaddress[doomcom->remotenode]`.

---

### `PacketGet`

```c
void PacketGet(void)
```

**Purpose:** Receives a UDP datagram into a local buffer, identifies which player node sent it by comparing the source address against `sendaddress[]`, and copies the byte-swapped result into `netbuffer`.

**Parameters:** None (operates on global `netbuffer` and `doomcom`).
**Returns:** Nothing. Sets `doomcom->remotenode = -1` if no packet is available or if the packet came from an unknown address.

**Key logic:**
- Calls `recvfrom` with `EWOULDBLOCK` tolerance (non-blocking receive).
- Scans `sendaddress[]` to match the sender's IP address; if not found, discards the packet.
- Byte-swaps received fields from network order to host order: `checksum` (32-bit), `angleturn` and `consistancy` (16-bit).
- Sets `doomcom->remotenode` and `doomcom->datalength` on success.
- Debug print on first received packet.

---

### `GetLocalAddress`

```c
int GetLocalAddress(void)
```

**Purpose:** Determines the local machine's IP address by resolving its hostname.

**Parameters:** None.
**Returns:** The local IPv4 address as a 32-bit integer (in network byte order).

**Key logic:** Calls `gethostname`, then `gethostbyname`, and returns the first address from `h_addr_list[0]`.

---

### `I_InitNetwork`

```c
void I_InitNetwork(void)
```

**Purpose:** Initializes the networking subsystem. For single-player, sets up a minimal `doomcom_t` with one local player. For networked games, parses the `-net` argument to determine the console player number and remote host list, resolves hostnames, creates and binds sockets.

**Parameters:** None.
**Returns:** Nothing.

**Key logic:**
1. Allocates and zero-initializes `doomcom`.
2. Checks for `-dup` (tic duplication 1–9) and `-extratic` parameters.
3. Checks for `-port` to override the default DOOM port.
4. If `-net` is absent: configures a single-player game (`netgame=false`, 1 player, 1 node) and returns.
5. If `-net` is present:
   - Sets `netsend = PacketSend`, `netget = PacketGet`, `netgame = true`.
   - Parses console player number (1-indexed argument after `-net`).
   - Iterates remaining arguments as hostnames or IP addresses (prefixed with `.` for raw IPs).
   - Resolves each hostname and fills `sendaddress[doomcom->numnodes]`.
   - Creates and binds the receive socket (`insocket`) with `FIONBIO` set for non-blocking I/O.
   - Creates the send socket (`sendsocket`).

---

### `I_NetCmd`

```c
void I_NetCmd(void)
```

**Purpose:** Dispatches a network command from the engine by calling either `netsend()` or `netget()` based on `doomcom->command`.

**Parameters:** None.
**Returns:** Nothing. Calls `I_Error` if `doomcom->command` is neither `CMD_SEND` nor `CMD_GET`.

**Key logic:** Simple switch on `doomcom->command`:
- `CMD_SEND` → `netsend()`
- `CMD_GET` → `netget()`

## Data Structures

The `doomdata_t` and `doomcom_t` structures are defined in `d_net.h`. This module uses:

- `doomcom_t* doomcom` — Global pointer to the network communication structure.
- `doomdata_t* netbuffer` — Global pointer to the shared network packet buffer.
- `struct sockaddr_in` — Standard BSD socket address structure.

## Dependencies

- `stdlib.h`, `string.h`, `stdio.h` — Standard C library.
- `sys/socket.h`, `netinet/in.h`, `arpa/inet.h` — BSD socket API.
- `errno.h`, `unistd.h`, `netdb.h`, `sys/ioctl.h` — POSIX system calls.
- `i_system.h` — `I_Error` for fatal error reporting.
- `d_event.h` — Event types (included but not directly used here).
- `d_net.h` — `doomcom_t`, `doomdata_t`, `netbuffer`, `doomcom`, `DOOMCOM_ID`, `CMD_SEND`, `CMD_GET`.
- `m_argv.h` — `M_CheckParm`, `myargc`, `myargv` for command-line parsing.
- `doomstat.h` — `netgame` flag.
- `i_net.h` — Own header.
