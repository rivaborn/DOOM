# File Overview

`DOOMNET.C` is the shared network utility module for the DOOM serial link driver (`SERSETUP`). It provides two critical responsibilities: command-line argument parsing, and the `LaunchDOOM` function that bootstraps the main DOOM executable with the network communication interface.

This file is part of the DOS-based serial network driver subsystem. When a player wants to play DOOM over a serial (null-modem or modem) connection, `SERSETUP.EXE` is run first. It sets up the serial port, negotiates which player is player 0 or 1, and then uses `LaunchDOOM` to spawn the DOOM executable as a child process, passing the shared `doomcom_t` structure's flat-memory address via a command-line argument.

The `doomcom_t` structure acts as the shared memory interface between the network driver (running as a TSR-like interrupt handler) and the DOOM game process. DOOM triggers a software interrupt (`int 0x60`-`0x66`) whenever it needs to send or receive a network packet. The interrupt handler (`NetISR`, defined in `SERSETUP.C`) then acts on the request using the serial port.

The file is compiled with `#define DOOM2`, selecting French-language strings from `SER_FRCH.H` for user-visible messages (the English counterpart is `SERSTR.H`, which is commented out).

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `doomcom_t` | `doomcom` | The shared communication structure between the network driver and the DOOM game process. Fields are filled in before `LaunchDOOM` is called, and DOOM reads the address of this structure from the `-net` command-line argument. |
| `int` | `vectorishooked` | Boolean flag. Set to `1` after a software interrupt vector has been successfully hooked for the network ISR. Used by `Error()` in `SERSETUP.C` to know whether the vector needs to be unhooked on abnormal exit. |
| `void interrupt (*olddoomvect)(void)` | `olddoomvect` | Pointer to the original interrupt handler that was installed at the chosen interrupt vector before `NetISR` replaced it. Saved so it can be restored on program exit. |

---

## Functions

### `CheckParm`

**Signature:**
```c
int CheckParm(char *check)
```

**Purpose:**
Searches the program's command-line argument list for a specific parameter string, using a case-insensitive comparison.

**Parameters:**
- `check` - The parameter string to search for (e.g., `"-vector"`, `"-dial"`, `"-com2"`).

**Return Value:**
Returns the index (1 to `myargc-1`) of the matching argument if found, or `0` if not present. The index can be used to read the value of the following argument (e.g., `myargv[p+1]`).

**Key Logic:**
Iterates from `i=1` to `myargc-1`, comparing each argument with `stricmp` (case-insensitive string compare). Returns immediately on the first match, or `0` after exhausting all arguments.

---

### `LaunchDOOM`

**Signature:**
```c
void LaunchDOOM(void)
```

**Purpose:**
Performs the final setup and launches the DOOM game executable as a child process, with the serial network driver active as an interrupt service routine (ISR).

**Parameters:**
None.

**Return Value:**
None (void). The function blocks until the DOOM child process exits (via `spawnv` with `P_WAIT`).

**Key Logic:**

1. **Set the magic identifier:** `doomcom.id` is set to `DOOMCOM_ID` (`0x12345678`). DOOM uses this to verify the structure is valid.

2. **Find a free interrupt vector:** Searches software interrupt vectors `0x60` through `0x66` for one that is either NULL or points to an `IRET` instruction (`0xcf`). This is the standard DOS technique for finding unused "user" interrupt vectors. If none is free, a warning is printed and `0x66` is used as a fallback.
   - If `-vector 0x<num>` was passed on the command line, that vector is used instead.

3. **Hook the interrupt:** The old vector is saved via `getvect`, and `NetISR` (the serial network interrupt service routine from `SERSETUP.C`) is installed via `setvect`. `vectorishooked` is set to `1`.

4. **Build the argument list:** The original `myargv` array is copied into `newargs`. Two additional arguments are appended:
   - `-net` - signals to DOOM that a network driver is present.
   - A decimal string representation of the flat (segment:offset converted to linear) memory address of `doomcom`. DOOM reads this address, casts it to `doomcom_t*`, and uses it to communicate with the driver via the hooked interrupt.

5. **Spawn DOOM:** Checks for `doom2.exe` first (since compiled with `#define DOOM2`), then falls back to `doom`. Uses `spawnv(P_WAIT, ...)` which blocks until DOOM exits.

6. **Print return message:** Announces that DOOM has returned.

---

## Data Structures

This file uses `doomcom_t` which is defined in `DOOMNET.H`. See the `DOOMNET.H` documentation for the full structure definition.

---

## Dependencies

| File | Reason |
|------|--------|
| `<stdio.h>` | `printf`, `sprintf`, `sscanf` |
| `<io.h>` | `access` (checking if `doom2.exe` exists) |
| `<stdlib.h>` | `exit` |
| `<string.h>` | `memcpy`, `stricmp` |
| `<process.h>` | `spawnv`, `P_WAIT` |
| `<dos.h>` | `getvect`, `setvect`, far pointer arithmetic |
| `doomnet.h` | `doomcom_t`, `DOOMCOM_ID`, `CMD_SEND`, `CMD_GET`, `NetISR` declaration, extern variable declarations |
| `ser_frch.h` | Localized (French) string constants for user-visible messages (`STR_WARNING`, `STR_COMM`, `STR_RETURNED`) |
