# File Overview

`doomstat.c` is the definition file for the global game state variables declared in `doomstat.h`. It provides the actual storage for a small core set of global variables that describe the "meta" state of the running game: which IWAD was loaded (game mode and mission), what language is selected, and whether any unofficial WAD files (PWADs) have been added.

The file is intentionally minimal. The vast majority of global state variables declared in `doomstat.h` are defined in other modules (primarily `g_game.c` for gameplay state and other subsystem `.c` files for their own state). `doomstat.c` only defines the three variables that do not belong to any specific subsystem and have no other natural home.

## Global Variables

| Type | Name | Initial Value | Description |
|------|------|---------------|-------------|
| `GameMode_t` | `gamemode` | `indetermined` | Identifies which IWAD is loaded: shareware, registered, commercial (DOOM 2), or retail (Ultimate DOOM). Set during startup in `d_main.c` by detecting which WAD file is present. |
| `GameMission_t` | `gamemission` | `doom` | Distinguishes between DOOM, DOOM 2, TNT, and Plutonia. Used to select mission-specific behavior (e.g., different episode structures). |
| `Language_t` | `language` | `english` | The currently active language for message strings. Set during startup; selects between English, French, and German string tables. |
| `boolean` | `modifiedgame` | (false, zero-initialized) | Set to true when any unofficial PWAD has been loaded alongside the IWAD. Used to disable demo recording/playback for modified games and display the "modified game" warning. |

## Functions

No functions are defined in this file.

## Data Structures

No new data structures are defined in this file. It uses `GameMode_t`, `GameMission_t`, `Language_t`, and `boolean` from `doomdef.h` and `doomtype.h`.

## Dependencies

| File | Reason |
|------|--------|
| `doomstat.h` | The header being implemented; declares all the externs that this file provides definitions for, and includes `doomdata.h`, `d_net.h`, and `d_player.h` to pull in the full type system. |

### Notes on the RCS ID

The file contains:
```c
static const char rcsid[] = "$Id: m_bbox.c,v 1.1 1997/02/03 22:45:10 b1 Exp $";
```
This references `m_bbox.c` rather than `doomstat.c`, indicating a copy-paste artifact from the original source preparation.
