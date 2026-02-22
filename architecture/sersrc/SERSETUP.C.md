# File Overview

`SERSETUP.C` is the main source file for the DOOM serial network driver (`SERSETUP.EXE`). It is the entry point and highest-level coordinator of the serial link network system. Its responsibilities encompass:

- Program entry (`main`) and command-line processing including response files.
- Baud rate selection and modem configuration.
- Serial port packet framing: the `ReadPacket`/`WritePacket` protocol for reliable byte-stream-to-packet conversion.
- Peer connection negotiation (`Connect`) to determine which machine is player 0 and player 1.
- Modem control: dialing (`Dial`), answering (`Answer`), and issuing AT commands (`ModemCommand`/`ModemResponse`).
- The `NetISR` interrupt service routine that DOOM calls to send and receive packets.
- Graceful and error-based shutdown (`Error`).

The driver is compiled with `#define DOOM2`, causing it to use French localized strings from `SER_FRCH.H`.

The overall flow is: `main` parses arguments, reads modem config if needed, calls `InitPort` (from `PORT.C`), optionally dials or answers, runs the `Connect` handshake to assign player numbers, then calls `LaunchDOOM` (from `DOOMNET.C`) which spawns DOOM.EXE. After DOOM exits, `Error(NULL)` is called for a clean exit.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `int` | `myargc` | Count of command-line arguments. Mirrors the C standard `argc`, but may be replaced when a response file is processed by `FindResponseFile`. |
| `char **` | `myargv` | Array of command-line argument strings. May be replaced with a dynamically allocated array if a response file is found. |
| `int` | `usemodem` | Boolean flag. Set to `true` by `ReadModemCfg` when a modem configuration file is read. Controls whether `Error` issues modem hangup commands before shutting down. |
| `char` | `startup[256]` | Modem initialization AT command string, read from `modem.cfg` (first line). |
| `char` | `shutdown[256]` | Modem hangup AT command string, read from `modem.cfg` (second line). |
| `char` | `baudrate[256]` | Baud rate string read from `modem.cfg` (third line), converted to a numeric divisor. |
| `char` | `packet[MAXPACKET]` | Buffer holding the most recently assembled complete incoming packet. `MAXPACKET` is 512 bytes. |
| `int` | `packetlen` | Length in bytes of the data currently in `packet[]`. |
| `int` | `inescape` | State flag for the packet framing state machine. `1` means the previous byte was a `FRAMECHAR` (0x70), so the next byte determines whether it is a literal `FRAMECHAR` (if also `0x70`) or a packet terminator (if anything else). |
| `int` | `newpacket` | State flag indicating that `ReadPacket` should reset and start assembling a fresh packet on the next call. Set to `1` after a complete packet is received or after a buffer overflow. |
| `char` | `response[80]` | Buffer for storing the last modem response line read by `ModemResponse`. |

---

## Functions

### `I_Error`

**Signature:**
```c
void I_Error(char *string)
```

**Purpose:**
Simple fatal error handler used internally by `FindResponseFile`. Prints a message and exits with status 1.

**Parameters:**
- `string` - The error message to print.

**Return Value:**
Does not return (calls `exit(1)`).

---

### `write_buffer`

**Signature:**
```c
void write_buffer(char *buffer, unsigned int count)
```

**Purpose:**
Writes a block of bytes into the transmit queue (`outque`) for serial transmission, and triggers the transmit hardware if necessary.

**Parameters:**
- `buffer` - Pointer to the bytes to transmit.
- `count` - Number of bytes to transmit.

**Return Value:**
None (void).

**Key Logic:**
1. **Overflow protection:** If the number of bytes already queued plus `count` would exceed `QUESIZE`, the entire outgoing queue is discarded (`outque.tail = outque.head`) before writing new data.
2. Calls `write_byte` (from `PORT.C`) for each byte.
3. Checks the UART Line Status Register bit 6 (`0x40`, the "transmitter empty" flag). If the transmitter is idle, calls `jump_start` (from `PORT.C`) to kick off the interrupt-driven transmission chain. Without this, the first byte would sit in the queue forever waiting for a TX interrupt that never comes.

---

### `Error`

**Signature:**
```c
void Error(char *error, ...)
```

**Purpose:**
The primary cleanup and exit function. Handles both graceful exits (when called with `NULL`) and error exits (when called with a format string). Performs all teardown in the correct order regardless.

**Parameters:**
- `error` - A `printf`-style format string for the error message, or `NULL` for a clean exit.
- `...` - Additional arguments for the format string.

**Return Value:**
Does not return.

**Key Logic:**
1. **Modem hangup:** If `usemodem` is true, drops DTR (clears `MCR_DTR` bit in the Modem Control Register), waits 1250ms, restores DTR, sends `+++` (Hayes escape sequence), waits again, then sends the `shutdown` command string (from `modem.cfg`).
2. Calls `ShutdownPort()` (from `PORT.C`) to disable UART interrupts and restore the IRQ vector.
3. If `vectorishooked`, restores the DOS software interrupt vector using `setvect`.
4. If `error` is non-NULL, uses `vprintf` to print the formatted message and exits with code 1.
5. If `error` is NULL, prints a clean exit message and exits with code 0.

---

### `ReadPacket`

**Signature:**
```c
boolean ReadPacket(void)
```

**Purpose:**
Reads bytes from the serial receive queue and assembles them into a complete packet using a byte-stuffing framing protocol. Returns `true` when a complete packet is available in `packet[]`.

**Parameters:**
None.

**Return Value:**
`true` (non-zero) if a complete packet has been received and is ready in `packet[]` with its length in `packetlen`. `false` if more data is still needed.

**Key Logic:**

The framing protocol uses `FRAMECHAR` (0x70) as a special delimiter:
- A single `FRAMECHAR` followed by a non-`FRAMECHAR` byte signals the **end of a packet** (the second byte is discarded as a terminator marker).
- Two consecutive `FRAMECHAR` bytes are an **escaped literal** 0x70 byte.

State machine:
1. **Buffer overflow check:** If `inque.head - inque.tail > QUESIZE - 4`, the queue is nearly full. Discard all data and reset `newpacket = true`.
2. **New packet reset:** If `newpacket` is set, resets `packetlen` to 0 and clears `newpacket`.
3. **Byte loop:** Calls `read_byte()` repeatedly until it returns `-1` (no more data):
   - If `inescape` is set: the previous byte was a `FRAMECHAR`. If this byte is also `FRAMECHAR`, it's a literal — clear `inescape` and fall through to store it. If it's anything else, the packet is complete: set `newpacket = 1`, return `true`.
   - If the current byte is `FRAMECHAR`: set `inescape = true`, `continue` (skip storing, wait for next byte to determine meaning).
   - If `packetlen >= MAXPACKET`: skip storing (oversize packet protection).
   - Otherwise: store the byte at `packet[packetlen++]`.

---

### `WritePacket`

**Signature:**
```c
void WritePacket(char *buffer, int len)
```

**Purpose:**
Transmits a packet over the serial link using the byte-stuffing framing protocol. Any `FRAMECHAR` (0x70) bytes within the data are escaped by doubling them. A two-byte `FRAMECHAR + 0x00` sequence is appended to mark the end of the packet.

**Parameters:**
- `buffer` - Pointer to the raw packet data to send.
- `len` - Length of the data in bytes.

**Return Value:**
None (void). Returns immediately without sending if `len > MAXPACKET`.

**Key Logic:**
1. Rejects packets larger than `MAXPACKET` (512 bytes).
2. Iterates through `buffer`, byte by byte:
   - If a byte equals `FRAMECHAR`, emits an extra `FRAMECHAR` first (escape sequence: `FRAMECHAR FRAMECHAR` = literal 0x70).
   - Always emits the byte itself.
3. Appends the end-of-packet terminator: `FRAMECHAR` followed by `0x00`.
4. Calls `write_buffer` to enqueue the entire framed packet for transmission.

Uses a static `localbuffer` of size `MAXPACKET*2+2` to accommodate worst-case escaping (every byte is a `FRAMECHAR`) plus the 2-byte terminator.

---

### `NetISR`

**Signature:**
```c
void interrupt NetISR(void)
```

**Purpose:**
The software interrupt service routine that DOOM calls (via `int <doomcom.intnum>`) to send or receive network packets. This is the bridge between the DOOM game engine and the serial port hardware.

**Parameters:**
None (interrupt handler).

**Return Value:**
None (void).

**Key Logic:**
Checks `doomcom.command`:

- **`CMD_SEND` (1):** Calls `WritePacket` with `doomcom.data` and `doomcom.datalength` to transmit the packet DOOM has prepared.
- **`CMD_GET` (2):** Calls `ReadPacket`. If a complete packet is available and its length fits in `doomcom.data`:
  - Sets `doomcom.remotenode = 1` (serial is always a two-player connection; the remote is node 1).
  - Sets `doomcom.datalength = packetlen`.
  - Copies `packet[]` into `doomcom.data`.
  - If no packet is ready, sets `doomcom.remotenode = -1` to signal "no packet".

---

### `Connect`

**Signature:**
```c
void Connect(void)
```

**Purpose:**
Negotiates a connection with the remote machine over the serial link to determine which machine is player 0 (consoleplayer 0) and which is player 1. This is necessary because the two machines must agree on roles before DOOM is launched.

**Parameters:**
None.

**Return Value:**
None (void). Sets `doomcom.consoleplayer` to 0 or 1.

**Key Logic:**

1. **ID generation:** Creates a 6-digit decimal string (`idstr`) that is (hopefully) unique per machine, derived from the current time in hundredths of a second plus a hash of the interrupt vector table. Alternatively, `-player1` forces `idnum=0` and `-player2` forces `idnum=999999` for deterministic assignment.

2. **Handshake protocol:** The packet format is `ID<6-digit-id>_<stage>`. The stage starts at 0 and increments as acknowledgement progresses:
   - Stage 0: "I'm here, my ID is X".
   - Stage 1: "I've seen your ID, I'm at stage 1".
   - Stage 2: "I've seen you reach stage 1, handshake complete".

   The loop sends the local ID packet once per second (`oldsec` timer), reads incoming packets, and updates `localstage = remotestage + 1` each time a valid remote packet is received. The loop exits when `localstage >= 2`.

3. **Duplicate detection:** If the remote ID string matches the local ID, an error is reported (extremely unlikely collision, but handled).

4. **Player assignment:** After the handshake, the machine with the lexicographically greater ID string becomes player 1; the other becomes player 0. This is a deterministic tie-breaking rule that both machines independently arrive at.

5. **Queue flush:** Drains any extra packets received during the handshake before returning.

---

### `ModemCommand`

**Signature:**
```c
void ModemCommand(char *str)
```

**Purpose:**
Sends an AT command string to the modem over the serial port, one character at a time with 100ms delays between each character, followed by a carriage return.

**Parameters:**
- `str` - The AT command string to send (e.g., `"ATZ"`, `"ATDT5551234"`).

**Return Value:**
None (void).

**Key Logic:**
Iterates through each character in `str`, calling `write_buffer` for each with a 100ms `delay()` between characters. Appends `"\r"` (carriage return) after the string. Also echoes everything to stdout.

---

### `ModemResponse`

**Signature:**
```c
void ModemResponse(char *resp)
```

**Purpose:**
Waits for the modem to send a response line that begins with the specified string (e.g., `"OK"`, `"CONNECT"`, `"RING"`).

**Parameters:**
- `resp` - The expected response prefix to wait for.

**Return Value:**
None (void). Blocks until the response is received.

**Key Logic:**
Outer loop continues until a response line starting with `resp` is found. Inner loop reads characters from `read_byte()` (blocking with escape-key check via `bioskey`) until a newline or 79 characters are accumulated. Printable characters are stored in `response[]`; control characters (below space) are discarded. The completed line is compared to `resp` using `strncmp`.

---

### `ReadLine`

**Signature:**
```c
void ReadLine(FILE *f, char *dest)
```

**Purpose:**
Reads a single line of text from an open file into a destination buffer, stopping at EOF, `\r`, or `\n`.

**Parameters:**
- `f` - Open file handle to read from.
- `dest` - Destination character buffer (must be large enough for the line).

**Return Value:**
None (void). Writes the line to `dest` as a null-terminated string.

---

### `ReadModemCfg`

**Signature:**
```c
void ReadModemCfg(void)
```

**Purpose:**
Reads the three-line `modem.cfg` file to configure modem initialization, hangup strings, and baud rate.

**Parameters:**
None.

**Return Value:**
None (void).

**Key Logic:**
Opens `modem.cfg` and reads three lines into `startup`, `shutdown`, and `baudrate` global arrays. Converts `baudrate` to a UART divisor (`baudbits = 115200 / baud`). Sets `usemodem = true`. Calls `Error` if the file cannot be opened.

---

### `Dial`

**Signature:**
```c
void Dial(void)
```

**Purpose:**
Initiates a modem call to the number specified by the `-dial <number>` command-line argument.

**Parameters:**
None.

**Return Value:**
None (void).

**Key Logic:**
1. Sends the `startup` AT command (from `modem.cfg`) and waits for `"OK"`.
2. Constructs `"ATDT<number>"` from the `-dial` argument and sends it.
3. Waits for a `"CONNECT"` response from the modem.
4. Sets `doomcom.consoleplayer = 1` (the dialing end is always player 1 by convention, before `Connect` may override this).

---

### `Answer`

**Signature:**
```c
void Answer(void)
```

**Purpose:**
Configures the modem to answer an incoming call.

**Parameters:**
None.

**Return Value:**
None (void).

**Key Logic:**
1. Sends the `startup` AT command and waits for `"OK"`.
2. Waits for a `"RING"` response.
3. Sends `"ATA"` (answer) and waits for `"CONNECT"`.
4. Sets `doomcom.consoleplayer = 0` (the answering end is player 0 by convention).

---

### `FindResponseFile`

**Signature:**
```c
void FindResponseFile(void)
```

**Purpose:**
Scans the command-line arguments for a response file argument (an argument beginning with `@`). If found, reads the file and replaces `myargv`/`myargc` with the combined argument list.

**Parameters:**
None.

**Return Value:**
None (void).

**Key Logic:**
Iterates through `myargv` looking for an argument starting with `@`. If found:
1. Opens the file named by the rest of the argument (after the `@`).
2. Reads the entire file into a malloc'd buffer.
3. Parses whitespace-separated tokens from the file as additional arguments (replaces `myargv` with a new malloc'd array capped at `MAXARGVS=100`).
4. Appends any arguments that followed the `@file` argument in the original command line.
5. Updates `myargc` to the new count.

---

### `main`

**Signature:**
```c
void main(void)
```

**Purpose:**
Program entry point. Orchestrates the complete startup sequence for the serial network driver.

**Parameters:**
None (uses DOS `_argc`/`_argv` globals directly).

**Return Value:**
None (void, exits via `Error`).

**Key Logic:**
1. Sets default `doomcom` fields: `ticdup=1`, `extratics=0`, `numnodes=2`, `numplayers=2`, `drone=0`.
2. Prints the driver banner.
3. Copies `_argc`/`_argv` to `myargc`/`myargv` and calls `FindResponseFile`.
4. Sets default `baudbits = 0x08` (9600 baud divisor).
5. If `-dial` or `-answer` is present, calls `ReadModemCfg` (which may set `baudbits` from the config file).
6. Processes baud-rate override arguments: `-9600`, `-14400`, `-19200`, `-38400`, `-57600`, `-115200`.
7. Calls `InitPort()` to configure the UART and install the ISR.
8. Calls `Dial()` or `Answer()` if a modem mode is selected.
9. Calls `Connect()` to perform the handshake and assign player numbers.
10. Calls `LaunchDOOM()` to spawn the DOOM executable.
11. Calls `Error(NULL)` for a clean exit after DOOM returns.

---

## Data Structures

No new structs are defined in this file. It uses `doomcom_t` from `DOOMNET.H`, `que_t` from `SERSETUP.H`, and `struct time` from `<dos.h>`.

---

## Preprocessor Constants (local)

| Constant | Value | Description |
|----------|-------|-------------|
| `MAXPACKET` | `512` | Maximum packet payload size in bytes |
| `FRAMECHAR` | `0x70` | Special framing byte used to delimit packets in the serial byte stream |
| `DOOM2` | (defined) | Selects DOOM II branding and French string table |
| `MAXARGVS` | `100` | Maximum number of arguments when expanding a response file |

---

## Dependencies

| File | Reason |
|------|--------|
| `sersetup.h` | `que_t`, UART register constants, `INPUT`/`OUTPUT` macros, `boolean` type, function declarations for `InitPort`/`ShutdownPort`/`read_byte`/`write_byte` |
| `ser_frch.h` | Localized string constants for user-visible output |
| `DoomNet.h` | `doomcom_t`, `doomcom`, `vectorishooked`, `olddoomvect`, `CMD_SEND`, `CMD_GET`, `LaunchDOOM`, `CheckParm` |
| `<stdio.h>` | `printf`, `vprintf`, `fopen`, `fclose`, `fgetc`, `fread`, `fseek`, `ftell`, `sprintf` |
| `<stdlib.h>` | `exit`, `malloc`, `memset`, `atol` |
| `<string.h>` | `memcpy`, `strlen`, `strncmp`, `strcmp`, `strncpy`, `strupr` |
| `<stdarg.h>` | `va_list`, `va_start`, `va_end` for variadic `Error` |
| `<dos.h>` | `delay`, `setvect`, `getvect`, `OUTPUT`/`INPUT` for UART registers |
| `<bios.h>` | `bioskey` for keyboard input checking during modem wait loops |
| `<process.h>` | (via `DoomNet.h`, used in `LaunchDOOM`) |
