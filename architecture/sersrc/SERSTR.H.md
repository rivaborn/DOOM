# File Overview

`SERSTR.H` is the English-language string table header for the DOOM serial network driver (`SERSETUP`). It defines all user-visible text strings as preprocessor macros, enabling the driver to be localized by simply swapping in a different string header.

The French equivalent of this file is `SER_FRCH.H`. In the released source, `SERSETUP.C`, `PORT.C`, and `DOOMNET.C` all include `SER_FRCH.H` and have the `#include "serstr.h"` line commented out, meaning the released binary was built as the French localized version.

This file contains no code, functions, data structures, or variables — only `#define` string constants.

---

## Global Variables

None.

---

## Functions

None.

---

## Data Structures

None.

---

## Preprocessor String Constants

Each macro expands to a string literal used in `printf`/`sprintf`/`Error` calls throughout the serial driver source files.

| Macro | String Value | Used In | Context |
|-------|-------------|---------|---------|
| `STR_DROPDTR` | `"Dropping DTR"` | `SERSETUP.C` / `Error()` | Printed before the modem hangup sequence begins |
| `STR_CLEANEXIT` | `"Clean exit from SERSETUP"` | `SERSETUP.C` / `Error(NULL)` | Printed on normal program termination |
| `STR_ATTEMPT` | `"Attempting to connect across serial link, press escape to abort."` | `SERSETUP.C` / `Connect()` | Displayed while waiting for the remote peer |
| `STR_NETABORT` | `"Network game synchronization aborted."` | `SERSETUP.C` / `Connect()` | Displayed when the user presses Escape during `Connect` |
| `STR_DUPLICATE` | `"Duplicate id string, try again or check modem init string."` | `SERSETUP.C` / `Connect()` | Displayed when both machines generate the same ID hash |
| `STR_MODEMCMD` | `"Modem command : "` | `SERSETUP.C` / `ModemCommand()` | Prefix printed before each AT command sent to the modem |
| `STR_MODEMRESP` | `"Modem response: "` | `SERSETUP.C` / `ModemResponse()` | Prefix printed while waiting for a modem response |
| `STR_RESPABORT` | `"Modem response aborted."` | `SERSETUP.C` / `ModemResponse()` | Displayed when the user presses Escape during a modem wait |
| `STR_CANTREAD` | `"Couldn't read MODEM.CFG"` | `SERSETUP.C` / `ReadModemCfg()` | Fatal error when `modem.cfg` cannot be opened |
| `STR_DIALING` | `"Dialing..."` | `SERSETUP.C` / `Dial()` | Status message while dialing |
| `STR_CONNECT` | `"CONNECT"` | `SERSETUP.C` / `Dial()` and `Answer()` | The modem response string to wait for after connecting |
| `STR_WAITRING` | `"Waiting for ring..."` | `SERSETUP.C` / `Answer()` | Status while waiting for an incoming call |
| `STR_RING` | `"RING"` | `SERSETUP.C` / `Answer()` | The modem response string indicating an incoming call |
| `STR_NORESP` | `"No such response file!"` | `SERSETUP.C` / `FindResponseFile()` | Fatal error when a `@responsefile` argument cannot be opened |
| `STR_DOOMSERIAL` | `"DOOM II SERIAL DEVICE DRIVER v1.4"` | `SERSETUP.C` / `main()` | Driver banner printed at startup |
| `STR_WARNING` | (multi-line) | `DOOMNET.C` / `LaunchDOOM()` | Warning printed when no free interrupt vector (0x60-0x66) is found |
| `STR_COMM` | `"Communicating with interrupt vector 0x%x"` | `DOOMNET.C` / `LaunchDOOM()` | Reports which software interrupt vector was chosen |
| `STR_RETURNED` | `"Returned from DOOM II"` | `DOOMNET.C` / `LaunchDOOM()` | Printed after DOOM exits |
| `STR_PORTSET` | `"Setting port to %lu baud"` | `PORT.C` / `InitPort()` | Reports configured baud rate |

### Multi-line Macro

`STR_WARNING` spans multiple lines using string literal concatenation:

```c
#define STR_WARNING \
"Warning: no NULL or iret interrupt vectors were found in the 0x60 to 0x66\n"\
"range.  You can specify a vector with the -vector 0x<num> parameter.\n"
```

Note: `SERSTR.H` does not define `STR_PORTLOOK`, `STR_UART8250`, `STR_UART16550`, or `STR_CLEARPEND` — these are only defined in `SER_FRCH.H`. This represents an incomplete localization: `PORT.C` always required the French header to compile successfully.

---

## Dependencies

None. This is a standalone header file containing only preprocessor macro definitions.

It is intended to be included by:
- `SERSETUP.C`
- `PORT.C`
- `DOOMNET.C`

In the released source, all three files include `SER_FRCH.H` instead, with the `#include "serstr.h"` line commented out.
