# File Overview

**File:** `ipx/IPXSTR.H`
**Target Platform:** MS-DOS (any; purely preprocessor definitions)

`IPXSTR.H` is the English-language string table for the DOOM IPX network driver. It defines all user-visible text strings as preprocessor macros, enabling the driver to be localized by swapping this header for a different language variant (specifically `IPX_FRCH.H` for French).

In the actual shipped code, `IPXSTR.H` is commented out in both `DOOMNET.C` and `IPXSETUP.C`, with `IPX_FRCH.H` (the French version) included instead. This file is preserved as the English reference.

---

## Global Variables

None. This is a header of preprocessor `#define` macros only.

---

## Functions

None.

---

## Data Structures

None.

---

## Preprocessor Definitions

All string macros defined in this file:

| Macro | English Text | Usage Context |
|-------|-------------|---------------|
| `STR_NETABORT` | `"Network game synchronization aborted."` | Printed when the user presses ESC during `LookForNodes()` to abort node discovery |
| `STR_UNKNOWN` | `"Got an unknown game packet during setup"` | Printed when a game-phase packet arrives from an unexpected source during setup |
| `STR_FOUND` | `"Found a node!"` | Printed each time a new network node is discovered during `LookForNodes()` |
| `STR_LOOKING` | `"Looking for a node"` | Printed while waiting for additional nodes to appear |
| `STR_MORETHAN` | `"More than %i players specified!"` | Error when more non-drone nodes are discovered than `MAXPLAYERS` allows |
| `STR_NONESPEC` | `"No players specified for game!"` | Error when all discovered nodes are drones (no human players) |
| `STR_CONSOLEIS` | `"Console is player %i of %i"` | Printed after player number assignment to show the local player number |
| `STR_NORESP` | `"No such response file!"` | Error when a `@responsefile` argument names a file that does not exist |
| `STR_FOUNDRESP` | `"Found response file"` | Status message when a response file is successfully opened |
| `STR_DOOMNETDRV` | `"DOOM II NETWORK DEVICE DRIVER"` | Driver banner title line (DOOM II variant) |
| `STR_VECTSPEC` | `"The specified vector (0x%02x) was already hooked."` | Error when the user manually specifies a `-vector` that is already in use by another handler |
| `STR_NONULL` | `"Warning: no NULL or iret interrupt vectors were found in the 0x60 to 0x66\nrange.  You can specify a vector with the -vector 0x<num> parameter."` | Warning when no free interrupt vector slot is found in the 0x60-0x66 range |
| `STR_COMMVECT` | `"Communicating with interrupt vector 0x%x"` | Status message reporting which interrupt vector was selected for DOOM communication |
| `STR_USEALT` | `"Using alternate port %i for network"` | Status message when the user overrides the default IPX socket with `-port` |
| `STR_RETURNED` | `"Returned from DOOM II"` | Printed after DOOM exits and control returns to the driver |
| `STR_ATTEMPT` | `"Attempting to find all players for %i player net play. Press ESC to exit.\n"` | Printed at the start of `LookForNodes()` to tell the user how many players are expected |

---

## Dependencies

| File | Reason |
|------|--------|
| None | This file has no `#include` directives |

**Included by:**
- `ipx/DOOMNET.C` (commented out; `ipx_frch.h` used instead)
- `ipx/IPXSETUP.C` (commented out; `ipx_frch.h` used instead)
