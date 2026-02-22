# File Overview

`SERSETUP.H` is the primary configuration and type-definition header for the DOOM serial port driver. It serves as the central header for the serial link subsystem, providing:

- Portable macros abstracting DOS I/O port access and interrupt control.
- A complete set of symbolic constants for all UART (Universal Asynchronous Receiver/Transmitter) register offsets and bit definitions, covering both the classic 8250 and the buffered 16550 variants.
- The `que_t` circular buffer structure used for interrupt-driven serial I/O.
- Type aliases and `boolean` enum for code clarity.
- Function prototypes for the serial port management functions defined in `PORT.C`.

This header is included by all `.C` files in the `sersrc` directory that need to interact with the serial hardware or the circular queues.

---

## Global Variables

None. This is a pure header file defining types, macros, and function prototypes only.

---

## Functions

The following function prototypes are declared in this header and defined in `PORT.C` or `SERSETUP.C`:

### `InitPort`

```c
void InitPort(void);
```

Initializes the UART hardware, determines the port I/O address and IRQ, sets the baud rate and line format (8N1), detects UART type (8250 vs 16550), clears pending interrupts, and installs the interrupt service routine.

---

### `ShutdownPort`

```c
void ShutdownPort(void);
```

Reverses the work of `InitPort`: disables UART interrupts, masks the IRQ in the PIC, restores the original IRQ vector, and resets the COM port to 9600 baud via BIOS.

---

### `read_byte`

```c
int read_byte(void);
```

Reads one byte from the receive circular queue. Returns the byte value (0-255) or `-1` if no data is available.

---

### `write_byte`

```c
void write_byte(int c);
```

Writes one byte into the transmit circular queue.

---

### `Error`

```c
void Error(char *error, ...);
```

Defined in `SERSETUP.C`. The primary error/exit handler. Performs modem hangup if needed, shuts down the serial port, unhooks the interrupt vector, prints an error message (or a clean exit message if `error` is `NULL`), and calls `exit`.

---

## Data Structures

### `que_t`

```c
#define QUESIZE 2048

typedef struct
{
    long            head, tail;         // head = next write position, tail = next read position
    unsigned char   data[QUESIZE];      // circular byte buffer
} que_t;
```

A circular (ring) buffer used for both the receive queue (`inque`) and the transmit queue (`outque`). The queue uses unsigned long indices that are allowed to grow monotonically (they never wrap); the actual array index is computed by `index & (QUESIZE-1)`. The queue is empty when `head == tail` and conceptually full when `head - tail >= QUESIZE`.

`QUESIZE` must be a power of 2 for the bitwise wrap trick to work correctly.

---

### `boolean` enum

```c
typedef enum { false, true } boolean;
```

A simple boolean type, predating C99's `_Bool`. Used as the return type of `ReadPacket` in `SERSETUP.C`.

---

### `byte` type alias

```c
typedef unsigned char byte;
```

An alias for `unsigned char`, used throughout the DOOM codebase for clarity.

---

## Preprocessor Constants

### I/O Port Abstraction Macros

| Macro | Expansion | Description |
|-------|-----------|-------------|
| `INPUT(port)` | `inportb(port)` | Read one byte from an I/O port (Borland C intrinsic) |
| `OUTPUT(port, data)` | `outportb(port, data)` | Write one byte to an I/O port (Borland C intrinsic) |
| `CLI()` | `disable()` | Disable hardware interrupts (Borland C intrinsic for the `CLI` x86 instruction) |
| `STI()` | `enable()` | Enable hardware interrupts (Borland C intrinsic for the `STI` x86 instruction) |

These macros provide a thin portability layer over Borland C library functions.

---

### UART Register Offsets (relative to UART base I/O address)

All offsets are added to the `uart` base address variable.

| Constant | Value | Description |
|----------|-------|-------------|
| `TRANSMIT_HOLDING_REGISTER` | `0x00` | Write: byte to transmit (when DLAB=0) |
| `RECEIVE_BUFFER_REGISTER` | `0x00` | Read: received byte (when DLAB=0) |
| `DIVISOR_LATCH_LOW` | `0x00` | Low byte of baud rate divisor (when DLAB=1) |
| `DIVISOR_LATCH_HIGH` | `0x01` | High byte of baud rate divisor (when DLAB=1) |
| `INTERRUPT_ENABLE_REGISTER` | `0x01` | Controls which UART events generate IRQs |
| `INTERRUPT_ID_REGISTER` | `0x02` | Read: identifies the highest-priority pending interrupt |
| `FIFO_CONTROL_REGISTER` | `0x02` | Write: controls the 16550 FIFO (write-only on 16550) |
| `LINE_CONTROL_REGISTER` | `0x03` | Frame format (word length, stop bits, parity) and DLAB control |
| `MODEM_CONTROL_REGISTER` | `0x04` | Controls DTR, RTS, OUT1, OUT2 signals |
| `LINE_STATUS_REGISTER` | `0x05` | Reports transmit/receive status and error conditions |
| `MODEM_STATUS_REGISTER` | `0x06` | Reports modem handshake line states |

---

### Interrupt Enable Register (IER) Bit Definitions

| Constant | Value | Description |
|----------|-------|-------------|
| `IER_RX_DATA_READY` | `0x01` | Enable interrupt when received data is available |
| `IER_TX_HOLDING_REGISTER_EMPTY` | `0x02` | Enable interrupt when transmit holding register empties |
| `IER_LINE_STATUS` | `0x04` | Enable interrupt on line status changes (framing error, etc.) |
| `IER_MODEM_STATUS` | `0x08` | Enable interrupt on modem status changes (CTS, DSR, etc.) |

---

### Interrupt Identification Register (IIR) Values

| Constant | Value | Description |
|----------|-------|-------------|
| `IIR_MODEM_STATUS_INTERRUPT` | `0x00` | Pending: modem status change |
| `IIR_TX_HOLDING_REGISTER_INTERRUPT` | `0x02` | Pending: transmit holding register empty |
| `IIR_RX_DATA_READY_INTERRUPT` | `0x04` | Pending: received data available |
| `IIR_LINE_STATUS_INTERRUPT` | `0x06` | Pending: line status change (highest priority) |

When bit 0 of the IIR is `1`, there is no interrupt pending (the above codes all have bit 0 = 0).

---

### FIFO Control Register (FCR) Bit Definitions (16550 only)

| Constant | Value | Description |
|----------|-------|-------------|
| `FCR_FIFO_ENABLE` | `0x01` | Enable the transmit and receive FIFOs |
| `FCR_RCVR_FIFO_RESET` | `0x02` | Reset and clear the receive FIFO |
| `FCR_XMIT_FIFO_RESET` | `0x04` | Reset and clear the transmit FIFO |
| `FCR_RCVR_TRIGGER_LSB` | `0x40` | LSB of receive FIFO interrupt trigger level |
| `FCR_RCVR_TRIGGER_MSB` | `0x80` | MSB of receive FIFO interrupt trigger level |
| `FCR_TRIGGER_01` | `0x00` | RX FIFO interrupt after 1 byte |
| `FCR_TRIGGER_04` | `0x40` | RX FIFO interrupt after 4 bytes |
| `FCR_TRIGGER_08` | `0x80` | RX FIFO interrupt after 8 bytes |
| `FCR_TRIGGER_14` | `0xC0` | RX FIFO interrupt after 14 bytes |

---

### Line Control Register (LCR) Bit Definitions

| Constant | Value | Description |
|----------|-------|-------------|
| `LCR_WORD_LENGTH_MASK` | `0x03` | Mask for word length bits |
| `LCR_WORD_LENGTH_SELECT_0` | `0x01` | Word length bit 0 |
| `LCR_WORD_LENGTH_SELECT_1` | `0x02` | Word length bit 1 |
| `LCR_STOP_BITS` | `0x04` | 1 = 2 stop bits, 0 = 1 stop bit |
| `LCR_PARITY_MASK` | `0x38` | Mask for all parity bits |
| `LCR_PARITY_ENABLE` | `0x08` | 1 = parity enabled |
| `LCR_EVEN_PARITY_SELECT` | `0x10` | 1 = even parity, 0 = odd parity |
| `LCR_STICK_PARITY` | `0x20` | Stick parity mode |
| `LCR_SET_BREAK` | `0x40` | Force break condition on TX |
| `LCR_DLAB` | `0x80` | Divisor Latch Access Bit: 1 = access baud rate divisor |

The driver initializes LCR to `0x03` for 8 data bits, no parity, 1 stop bit (8N1).

---

### Modem Control Register (MCR) Bit Definitions

| Constant | Value | Description |
|----------|-------|-------------|
| `MCR_DTR` | `0x01` | Data Terminal Ready signal |
| `MCR_RTS` | `0x02` | Request To Send signal |
| `MCR_OUT1` | `0x04` | Auxiliary output 1 (general purpose) |
| `MCR_OUT2` | `0x08` | Auxiliary output 2 — must be set on ISA to enable IRQ from UART |
| `MCR_LOOPBACK` | `0x10` | Loopback test mode |

---

### Line Status Register (LSR) Bit Definitions

| Constant | Value | Description |
|----------|-------|-------------|
| `LSR_DATA_READY` | `0x01` | Received data is available in RBR |
| `LSR_OVERRUN_ERROR` | `0x02` | Receive buffer overrun error |
| `LSR_PARITY_ERROR` | `0x04` | Parity error on received byte |
| `LSR_FRAMING_ERROR` | `0x08` | Framing error on received byte |
| `LSR_BREAK_DETECT` | `0x10` | Break signal detected |
| `LSR_THRE` | `0x20` | Transmit Holding Register Empty — safe to write next byte |

---

### Modem Status Register (MSR) Bit Definitions

| Constant | Value | Description |
|----------|-------|-------------|
| `MSR_DELTA_CTS` | `0x01` | CTS has changed state |
| `MSR_DELTA_DSR` | `0x02` | DSR has changed state |
| `MSR_TERI` | `0x04` | Trailing Edge Ring Indicator |
| `MSR_DELTA_CD` | `0x08` | Carrier Detect has changed state |
| `MSR_CTS` | `0x10` | Clear To Send (current state) |
| `MSR_DSR` | `0x20` | Data Set Ready (current state) |
| `MSR_RI` | `0x40` | Ring Indicator |
| `MSR_CD` | `0x80` | Carrier Detect |

---

## Dependencies

| File | Reason |
|------|--------|
| `<conio.h>` | Included twice (redundant) — provides console I/O functions |
| `<stdio.h>` | Standard I/O |
| `<stdlib.h>` | Standard library |
| `<string.h>` | String functions |
| `<ctype.h>` | Character classification; included twice (redundant) |
| `<dos.h>` | DOS-specific types and BIOS/interrupt functions |
| `<process.h>` | Process management (`spawnv`) |
| `<stdarg.h>` | Variable argument lists |
| `<bios.h>` | BIOS keyboard functions (`bioskey`) |
