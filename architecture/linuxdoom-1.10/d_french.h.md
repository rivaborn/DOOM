# File Overview

`d_french.h` is the French-language localisation counterpart to `d_englsh.h`. It redefines every user-visible text string in DOOM using French translations. The file is selected by `dstrings.h` when the preprocessor symbol `FRENCH` is defined at compile time (typically passed as `-DFRENCH` to the compiler). This file contains only preprocessor `#define` macros — no code, variables, or type definitions.

The French version was created for the official French retail release. It covers the same string categories as `d_englsh.h` but omits some content that was never translated: notably the Plutonia/TNT mission pack level names (`PHUSTR_*`, `THUSTR_*`) and many of the DOOM II episode text screens beyond episode 3.

---

## Global Variables

None. Header-only preprocessor constants.

---

## Functions

None defined in this file.

---

## Data Structures

None defined in this file.

---

## String Categories

### D_Main strings
| Macro | French value |
|-------|-------------|
| `D_DEVSTR` | `"MODE DEVELOPPEMENT ON.\n"` |
| `D_CDROM` | `"VERSION CD-ROM: DEFAULT.CFG DANS C:\\DOOMDATA\n"` |

### M_Menu strings
French translations of all menu prompts, confirmations, and error messages:
- `PRESSKEY`, `PRESSYN` — generic prompts.
- `QUITMSG`, `LOADNET`, `QLOADNET`, `QSAVESPOT`, `SAVEDEAD` — error/quit messages.
- `QSPROMPT`, `QLPROMPT` — quick-save/load confirmations.
- `NEWGAME`, `NIGHTMARE`, `SWSTRING` — game-mode restriction messages.
- `MSGOFF`, `MSGON`, `NETEND`, `ENDGAME`, `DOSY` — miscellaneous UI strings.
- `DETAILHI`, `DETAILLO`, `GAMMALVL0`–`GAMMALVL4`, `EMPTYSTRING` — setting labels.

### P_inter strings
French item pickup messages for all armour, health, key, power-up, ammo, and weapon items.

Notable differences from English:
- Key pickup strings use "CARTE MAGNETIQUE" for keycards and "CLEF CRANE" for skull keys.
- Door-lock messages (`PD_BLUEK`, `PD_REDK`, `PD_YELLOWK`) are aliased to their object-activation equivalents (`PD_BLUEO`, etc.) rather than having separate translations.

### G_game string
- `GGSAVED` = `"JEU SAUVEGARDE."`

### HU_stuff strings
- HUD level names for DOOM 1 episodes 1–3 only (E1M1–E3M9); episode 4 names are absent.
- DOOM 2 level names (HUSTR_1–HUSTR_32) translated into French.
- Chat macros (`HUSTR_CHATMACRO0`–`9`) translated.
- `HUSTR_TALKTOSELF1`–`5` translated.
- Player colour prefixes: `HUSTR_PLRGREEN` = `"VERT: "`, etc.
- Key shortcuts retain the same Latin letters (`'g'`, `'i'`, `'b'`, `'r'`) with a comment noting the French key should be `'V'`.

### AM_map strings
Automap status messages in French:
- `AMSTR_FOLLOWON` = `"MODE POURSUITE ON"`
- `AMSTR_FOLLOWOFF` = `"MODE POURSUITE OFF"`
- `AMSTR_GRIDON` = `"GRILLE ON"`, `AMSTR_GRIDOFF` = `"GRILLE OFF"`
- `AMSTR_MARKEDSPOT` = `"REPERE MARQUE "`, `AMSTR_MARKSCLEARED` = `"REPERES EFFACES "`

### ST_stuff strings
Status bar cheat feedback strings translated into French.

### F_Finale strings
Episode ending text screens for DOOM 1 episodes 1–3 (`E1TEXT`, `E2TEXT`, `E3TEXT`) and DOOM 2 chapters 1–4 (`C1TEXT`–`C4TEXT`) and secret level notices (`C5TEXT`, `C6TEXT`). Episode 4 text (`E4TEXT`) is absent in the French version.

Plutonia and TNT text screens (`P*TEXT`, `T*TEXT`) are not present in this file.

### F_FINALE cast strings
Monster and player names for the DOOM 2 cast parade translated into French:
- `CC_ZOMBIE` = `"ZOMBIE"`, `CC_SHOTGUN` = `"TYPE AU FUSIL"`, etc.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `dstrings.h` | Includes this file when `FRENCH` is defined at compile time. |

All content is preprocessor text macros; no runtime dependencies.
