# File Overview

**File:** `ipx/DOOMNET.C`
**Target Platform:** MS-DOS (Borland C / Watcom C, 16-bit real mode)

`DOOMNET.C` in the IPX directory is a shared utility module that handles two critical responsibilities for the DOOM IPX network driver: launching the DOOM executable as a child process with the correct network parameters, and providing the interrupt service routine (ISR) hook that the running DOOM process uses to communicate back to the driver at runtime.

The file acts as the bridge between the setup/driver process (IPXSETUP) and the DOOM game itself. DOOM communicates with its network driver through a software interrupt mechanism: DOOM fires a CPU interrupt whenever it needs to send or receive a network packet, and this file installs the ISR that handles those interrupts by dispatching into the IPX send/receive functions.

The `#define DOOM2` at the top (commented out) reveals that the same codebase was used to build both the DOOM and DOOM II IPX drivers, with conditional compilation controlling which executable name and display strings are used.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `doomcom_t` | `doomcom` | The central communication structure shared between the driver and the DOOM executable. Allocated in the driver's data segment and passed to DOOM via its flat (segmented) address. DOOM reads game configuration from it and uses the `command`, `remotenode`, and `datalength` fields to issue send/receive requests through the interrupt vector. |
| `int` | `vectorishooked` | Boolean flag. Set to `1` after `setvect()` has been called to install `NetISR` into the interrupt table. Used by the `Error()` function in `IPXSETUP.C` to know whether the old vector must be restored before exiting. |
| `void interrupt (*olddoomvect)(void)` | `olddoomvect` | Pointer to the interrupt handler that previously occupied the interrupt slot chosen for DOOM communication. Saved before hooking so it can be restored on clean exit or error. |

---

## Functions

### `LaunchDOOM`

**Signature:**
```c
void LaunchDOOM(void)
```

**Purpose:**
Prepares the `doomcom` structure with its magic ID, hooks the interrupt vector so DOOM can reach the driver's `NetISR`, then builds a new argument list for `doom.exe` or `doom2.exe` that appends `-net <flat_address>` pointing to the `doomcom` structure. Launches DOOM as a synchronous child process using `spawnv(P_WAIT, ...)` and waits for it to exit.

**Parameters:** None.

**Return Value:** `void`. Returns after DOOM has exited.

**Key Logic:**

1. **Set the magic ID:** `doomcom.id = DOOMCOM_ID` (`0x12345678`). DOOM checks this value when it receives the `-net` argument to confirm the structure is valid.

2. **Hook the interrupt vector:**
   - Calls `getvect(doomcom.intnum)` to save the current handler into `olddoomvect`.
   - Calls `setvect(doomcom.intnum, MK_FP(_CS, (int)NetISR))` to install `NetISR`. `MK_FP` constructs a far pointer from the current code segment (`_CS`) and the offset of `NetISR`.
   - Sets `vectorishooked = 1`.

3. **Build argument list:** Copies the driver's own `_argv` array into `newargs`, then appends two extra arguments: the string `"-net"` and a decimal string representation of the flat (linear) address of `doomcom`. The flat address is computed as `(long)_DS * 16 + (unsigned)&doomcom`, converting the real-mode segmented address to a 20-bit linear address.

4. **Spawn DOOM:** Checks for the presence of `doom2.exe` using `access()`. If found, spawns `doom2`; otherwise spawns `doom`. Both use `P_WAIT` so the driver blocks until DOOM exits.

5. **Print return message:** After DOOM exits, prints a return message (language-dependent via the string header include).

---

## Data Structures

This file does not define any structs. It uses `doomcom_t` defined in `doomnet.h`.

---

## Dependencies

| File | Reason |
|------|--------|
| `<stdio.h>` | `printf`, `sprintf` |
| `<stdlib.h>` | `access` |
| `<string.h>` | `memcpy` |
| `<process.h>` | `spawnv`, `_argc`, `_argv` |
| `<conio.h>` | Console I/O |
| `<dos.h>` | `getvect`, `setvect`, `MK_FP`, `_CS`, `_DS` |
| `doomnet.h` | `doomcom_t`, `DOOMCOM_ID`, `CMD_SEND`, `CMD_GET`, `NetISR` declaration, `vectorishooked`, `olddoomvect` |
| `ipx_frch.h` (or `ipxstr.h`) | Localized string macros such as `STR_RETURNED`, `STR_DOOMNETDRV` |

The `NetISR` function referenced in `setvect()` is defined in `IPXSETUP.C`, not in this file.
