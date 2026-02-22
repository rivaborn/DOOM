# File Overview

`dstrings.h` is the header for DOOM's global string resources. It pulls in the appropriate language-specific string definitions (English by default, or French when compiled with `-DFRENCH`), defines miscellaneous string constants for file paths and save game names, sets the quit message count, and declares the `endmsg` quit message array.

The header acts as a language-selection layer: code that needs game strings includes `dstrings.h` rather than directly including `d_englsh.h` or `d_french.h`, allowing a single compile-time flag to switch the entire user-visible string table.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `char*[]` | `endmsg` | Declared `extern`; defined in `dstrings.c`. The array of quit confirmation messages, randomly selected when the player attempts to exit. Has `NUM_QUITMESSAGES + 1` entries. |

## Functions

No functions are declared in this header.

## Data Structures

No struct or enum types are defined.

## Defined Constants

### Language-Conditional Includes

```c
#ifdef FRENCH
#include "d_french.h"
#else
#include "d_englsh.h"
#endif
```

This conditional compilation block selects the entire game string table based on the `FRENCH` preprocessor define (passed via `-DFRENCH` at compile time). All message strings (level names, game messages, status bar text, etc.) are `#define` macros in these files.

### File and Path Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `SAVEGAMENAME` | `"doomsav"` | Base filename for save game files. Save files are named `doomsav0.dsg` through `doomsav5.dsg`. |
| `DEVMAPS` | `"devmaps"` | Relative path to development-mode map directory. Used in developer mode (`-devparm`). |
| `DEVDATA` | `"devdata"` | Relative path to development-mode data directory. |

### Quit Message Count

| Constant | Value | Description |
|----------|-------|-------------|
| `NUM_QUITMESSAGES` | `22` | The number of quit messages in the `endmsg` array (not counting the debug placeholder at the end). |

## Dependencies

| File | Reason |
|------|--------|
| `d_englsh.h` | Included by default; defines all English game message strings as `#define` macros. |
| `d_french.h` | Included when `FRENCH` is defined; provides the French string table. |
