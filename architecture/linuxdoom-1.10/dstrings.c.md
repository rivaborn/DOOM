# File Overview

`dstrings.c` defines the `endmsg` array — the collection of quit messages displayed when the player attempts to exit DOOM. These messages appear in the quit confirmation dialog, randomly selected each time the player tries to quit. The file serves as the translation unit that provides the concrete array definition that `dstrings.h` declares as `extern char* endmsg[]`.

The messages are grouped into three sets corresponding to different game versions: DOOM 1, DOOM 2, and FinalDOOM. The array ends with an internal debug/placeholder message. The total count is `NUM_QUITMESSAGES + 1` (22 + 1 = 23 entries, with the last being a blank-page marker).

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `char*[NUM_QUITMESSAGES+1]` | `endmsg` | Array of quit message string pointers. Indexed and randomly selected when the player quits. The last entry is an internal debug placeholder. |

**Contents of `endmsg`:**

| Index | Group | Message |
|-------|-------|---------|
| 0 | DOOM 1 | `QUITMSG` (defined in `d_englsh.h`/`d_french.h`) |
| 1 | DOOM 1 | "please don't leave, there's more\ndemons to toast!" |
| 2 | DOOM 1 | "let's beat it -- this is turning\ninto a bloodbath!" |
| 3 | DOOM 1 | "i wouldn't leave if i were you.\ndos is much worse." |
| 4 | DOOM 1 | "you're trying to say you like dos\nbetter than me, right?" |
| 5 | DOOM 1 | "don't leave yet -- there's a\ndemon around that corner!" |
| 6 | DOOM 1 | "ya know, next time you come in here\ni'm gonna toast ya." |
| 7 | DOOM 1 | "go ahead and leave. see if i care." |
| 8 | DOOM 2 | "you want to quit?\nthen, thou hast lost an eighth!" |
| 9 | DOOM 2 | "don't go now, there's a \ndimensional shambler waiting\nat the dos prompt!" |
| 10 | DOOM 2 | "get outta here and go back\nto your boring programs." |
| 11 | DOOM 2 | "if i were your boss, i'd \n deathmatch ya in a minute!" |
| 12 | DOOM 2 | "look, bud. you leave now\nand you forfeit your body count!" |
| 13 | DOOM 2 | "just leave. when you come\nback, i'll be waiting with a bat." |
| 14 | DOOM 2 | "you're lucky i don't smack\nyou for thinking about leaving." |
| 15–21 | FinalDOOM | Explicit/crude messages; notable for their unfiltered tone. |
| 22 | Debug | "THIS IS NO MESSAGE!\nPage intentionally left blank." |

**Note on indexing:** There is a subtle C syntax artifact: entries at indices 7 and 14 lack a trailing comma before the next string literal, causing those two strings to be concatenated with the following one at compile time. This means the array actually has fewer entries than intended for those groups. This appears to be a bug in the original source.

## Functions

No functions are defined in this file.

## Data Structures

No new data structures are defined.

## Dependencies

| File | Reason |
|------|--------|
| `dstrings.h` | The header this file implements; provides `NUM_QUITMESSAGES` count and the `extern char* endmsg[]` declaration. Also includes either `d_englsh.h` or `d_french.h` for the `QUITMSG` macro used in `endmsg[0]`. |
