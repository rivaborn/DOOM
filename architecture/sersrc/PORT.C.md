# File Overview

`PORT.C` implements the low-level serial port (UART) hardware management for the DOOM serial network driver. It handles:

- Detection and configuration of the serial port's I/O address and IRQ line.
- Initialization of the UART (both 8250 and 16550 variants).
- Installation and removal of the serial interrupt service routine (ISR).
- Interrupt-driven transmit and receive using circular queues.
- Initiating transmission via the "jump start" technique.

This file operates entirely at the hardware register level using direct I/O port reads and writes. It supports both ISA and MCA (Micro Channel Architecture) bus topologies and can detect the UART type to take advantage of the 16550's hardware FIFO for improved throughput.

The two ISR functions (`isr_8250` and `isr_16550`) are at the heart of this module. They run in interrupt context whenever the UART has data to transmit or has received data, and they manage the circular byte queues (`inque` and `outque`) that decouple the ISR from the main program logic.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `union REGS` | `regs` | DOS real-mode register structure used for BIOS interrupt calls (`int86`, `int86x`). |
| `struct SREGS` | `sregs` | DOS segment register structure used alongside `regs` for BIOS calls that return far pointers. |
| `que_t` | `inque` | Circular receive queue. Bytes arriving from the serial port are stored here by the ISR and consumed by `read_byte()`. |
| `que_t` | `outque` | Circular transmit queue. Bytes to be sent are placed here by `write_byte()` and consumed by the transmit ISR. |
| `int` | `uart` | The I/O base address of the UART (e.g., `0x3F8` for COM1). All UART register accesses are relative to this address. |
| `enum {UART_8250, UART_16550}` | `uart_type` | Detected UART chip type. Determines which ISR is installed and whether the FIFO is used. |
| `int` | `irq` | The IRQ number (e.g., 4 for COM1, 3 for COM2) associated with the selected serial port. |
| `int` | `modem_status` | Last-read value of the UART Modem Status Register. Initialized to `-1`; updated by the ISR when a modem status interrupt fires. |
| `int` | `line_status` | Last-read value of the UART Line Status Register. Initialized to `-1`; updated by the ISR when a line status interrupt fires. |
| `void interrupt (*oldirqvect)(void)` | `oldirqvect` | Saved pointer to the original IRQ interrupt handler, restored by `ShutdownPort`. |
| `int` | `irqintnum` | The interrupt vector number corresponding to the IRQ (`irq + 8`). COM1 IRQ4 maps to interrupt vector `0x0C`. |
| `int` | `comport` | Selected COM port number (1-4), determined by command-line arguments (`-com2`, `-com3`, `-com4`) or defaulting to 1. |
| `int` | `baudbits` | The baud rate divisor for the UART's Divisor Latch. `115200 / baud_rate` gives this value (e.g., `0x0C` for 9600, `0x01` for 115200). Set by `SERSETUP.C` before `InitPort` is called. |

---

## Functions

### `GetUart`

**Signature:**
```c
void GetUart(void)
```

**Purpose:**
Determines the I/O base address and IRQ number for the selected COM port by querying the system BIOS, and allows command-line overrides.

**Parameters:**
None.

**Return Value:**
None (void). Sets the global variables `comport`, `uart`, and `irq`.

**Key Logic:**

1. **COM port selection:** Checks command-line arguments `-com2`, `-com3`, `-com4`. Defaults to COM1 if none are present.

2. **Bus type detection:** Calls BIOS interrupt `0x15`, function `0xC0` ("Return System Configuration Parameters"). If the call fails (carry flag set), assumes ISA bus and uses the standard ISA UART addresses and IRQs:
   - COM1: `0x3F8`, IRQ 4
   - COM2: `0x2F8`, IRQ 3
   - COM3: `0x3E8`, IRQ 4
   - COM4: `0x2E8`, IRQ 3

3. **MCA detection:** If the BIOS call succeeds, checks bit 1 of the returned system descriptor byte to detect MCA bus. MCA uses different addresses for COM3/COM4:
   - COM3: `0x3220`, IRQ 3
   - COM4: `0x3228`, IRQ 3

4. **Command-line overrides:** `-port 0x<hex>` overrides the I/O address; `-irq <num>` overrides the IRQ number.

5. Prints the detected address and IRQ to stdout.

---

### `InitPort`

**Signature:**
```c
void InitPort(void)
```

**Purpose:**
Fully initializes the selected serial UART for interrupt-driven operation at the configured baud rate, installs the appropriate ISR, and enables interrupts.

**Parameters:**
None.

**Return Value:**
None (void).

**Key Logic:**

1. Calls `GetUart()` to determine the port address and IRQ.
2. Disables all UART interrupts by writing `0` to the Interrupt Enable Register.
3. Sets the baud rate by:
   - Setting bit 7 (DLAB) of the Line Control Register to enable access to the Divisor Latch.
   - Writing the low byte of `baudbits` to the Divisor Latch Low register, and `0` to Divisor Latch High.
   - Clearing DLAB by writing `0x03` (8N1 frame format: 8 data bits, no parity, 1 stop bit) to the Line Control Register.
4. Sets the Modem Control Register to `OUT2 | RTS | DTR` (`0x0B`) to enable the IRQ output and assert handshake lines.
5. **UART type detection:** Unless `-8250` is on the command line, attempts to enable the 16550 FIFO with a 4-byte trigger. Then reads the Interrupt Identification Register — if bits 7:3 are `0xC0`, it's a 16550; otherwise it's an 8250 and the FIFO is disabled.
6. **Pending interrupt flush:** Reads the Receive Buffer Register 16 times (to drain any 16550 FIFO), then loops reading the IIR until `bit 0` is set (indicating no more pending interrupts), servicing each pending interrupt type (modem status, line status, TX, RX).
7. **IRQ vector installation:** Saves the old IRQ vector and installs either `isr_16550` or `isr_8250`.
8. **Enable IRQ in PIC:** Clears the corresponding bit in the 8259 Programmable Interrupt Controller's Interrupt Mask Register (I/O port `0x21`) to unmask the IRQ.
9. Disables interrupts (`CLI()`), enables RX and TX interrupts in the UART's IER, sends the Non-Specific EOI command to the PIC (`0x20` to port `0x20`), then re-enables interrupts (`STI()`).

---

### `ShutdownPort`

**Signature:**
```c
void ShutdownPort(void)
```

**Purpose:**
Restores the serial port to a clean state: disables UART interrupts, re-masks the IRQ, restores the original interrupt vector, and reinitializes the port to 9600 baud defaults via BIOS.

**Parameters:**
None.

**Return Value:**
None (void).

**Key Logic:**

1. Writes `0` to the UART's Interrupt Enable Register to stop all UART interrupts.
2. Writes `0` to the Modem Control Register to deassert DTR, RTS, and OUT2.
3. Drains the 16550 receive FIFO by reading the Receive Buffer Register 16 times.
4. Re-masks the IRQ in the 8259 PIC by setting the appropriate bit in port `0x21`.
5. Restores the original IRQ vector via `setvect`.
6. Calls BIOS interrupt `0x14` with `AX=0xF3` (9600 baud, No parity, 8 bits, 1 stop) and `DX=comport-1` to reset the port to safe defaults.

---

### `read_byte`

**Signature:**
```c
int read_byte(void)
```

**Purpose:**
Reads and returns the next byte from the receive queue (`inque`), or returns `-1` if no data is available.

**Parameters:**
None.

**Return Value:**
The next received byte as an `int` (0-255), or `-1` if the queue is empty.

**Key Logic:**
Checks if `inque.tail >= inque.head` (empty condition). If empty, returns `-1`. Otherwise, extracts the byte at `inque.data[inque.tail & (QUESIZE-1)]`, increments `tail`, and returns the byte. The bitwise AND with `(QUESIZE-1)` implements the circular buffer wrap.

---

### `write_byte`

**Signature:**
```c
void write_byte(int c)
```

**Purpose:**
Appends a single byte to the transmit queue (`outque`) for eventual transmission by the transmit ISR.

**Parameters:**
- `c` - The byte value to enqueue for transmission.

**Return Value:**
None (void).

**Key Logic:**
Stores the byte at `outque.data[outque.head & (QUESIZE-1)]` and increments `outque.head`. Does not check for overflow (overflow is handled at a higher level in `write_buffer`).

---

### `isr_8250`

**Signature:**
```c
void interrupt isr_8250(void)
```

**Purpose:**
Hardware interrupt service routine for the 8250 UART (or 16550 operating in non-FIFO mode). Services all pending UART interrupt conditions in a single pass.

**Parameters:**
None (interrupt handler, called by hardware).

**Return Value:**
None (void).

**Key Logic:**
Loops continuously, reading the Interrupt Identification Register (IIR) to determine the interrupt source:

- **`IIR_MODEM_STATUS_INTERRUPT` (0x00):** Reads the Modem Status Register to acknowledge.
- **`IIR_LINE_STATUS_INTERRUPT` (0x06):** Reads the Line Status Register to acknowledge.
- **`IIR_TX_HOLDING_REGISTER_INTERRUPT` (0x02):** If `outque` has data, dequeues one byte and writes it to the Transmit Holding Register. The UART will generate another TX interrupt when ready for the next byte.
- **`IIR_RX_DATA_READY_INTERRUPT` (0x04):** Reads one byte from the Receive Buffer Register and enqueues it in `inque`.
- **Default (bit 0 set = no interrupt pending):** Sends a Non-Specific EOI (`0x20`) to the 8259 PIC and returns.

---

### `isr_16550`

**Signature:**
```c
void interrupt isr_16550(void)
```

**Purpose:**
Hardware interrupt service routine optimized for the 16550 UART's hardware FIFO. Identical in structure to `isr_8250` but takes advantage of the FIFO to process up to 16 bytes per TX interrupt and drain the entire RX FIFO in one pass.

**Parameters:**
None (interrupt handler, called by hardware).

**Return Value:**
None (void).

**Key Logic:**
Same overall loop structure as `isr_8250`, with two key differences:

- **TX interrupt:** Sends up to 16 bytes per interrupt (matching the 16550 FIFO depth) by looping `while (outque.tail < outque.head && count--)`.
- **RX interrupt:** Loops reading bytes from the Receive Buffer Register as long as the Line Status Register's `LSR_DATA_READY` bit remains set, draining the entire FIFO in one ISR invocation.

---

### `jump_start`

**Signature:**
```c
void jump_start(void)
```

**Purpose:**
Initiates the transmit interrupt chain when the UART's transmit holding register is empty and idle, preventing a deadlock where the ISR never fires because no transmission is in progress.

**Parameters:**
None.

**Return Value:**
None (void).

**Key Logic:**
The transmit ISR only fires when the UART is ready for more data — but if no transmission is in progress, the UART never generates the initial interrupt. `jump_start` solves this by manually writing the first byte directly to the UART's transmit register (I/O address `uart + 0`), which starts the hardware transmit sequence. Subsequent bytes will be sent by `isr_8250` or `isr_16550` as the TX interrupts fire. Called from `write_buffer` in `SERSETUP.C` when the Line Status Register indicates the transmit register is empty (`LSR_THRE`, bit 6 set).

---

## Data Structures

The `que_t` structure is defined in `SERSETUP.H`:

```c
typedef struct
{
    long            head, tail;         // head = write index, tail = read index
    unsigned char   data[QUESIZE];      // circular byte buffer (QUESIZE = 2048)
} que_t;
```

`inque` and `outque` are instances of this type, serving as the receive and transmit circular buffers respectively.

---

## Dependencies

| File | Reason |
|------|--------|
| `doomnet.h` | `doomcom_t`, `CheckParm`, `I_ColorBlack` debug macro |
| `sersetup.h` | `que_t` type definition, `QUESIZE` constant, UART register offset macros, `INPUT`/`OUTPUT` macros, `CLI`/`STI` macros, `InitPort`/`ShutdownPort` declarations |
| `ser_frch.h` | Localized string constants (`STR_PORTLOOK`, `STR_PORTSET`, `STR_UART8250`, `STR_UART16550`, `STR_CLEARPEND`) |
| DOS BIOS (INT 0x15) | System configuration query for bus type detection |
| DOS BIOS (INT 0x14) | Serial port reset to defaults in `ShutdownPort` |
