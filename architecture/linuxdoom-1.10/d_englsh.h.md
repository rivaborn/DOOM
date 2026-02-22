# File Overview

`d_englsh.h` is a pure-macro header containing every user-visible text string for the English-language version of DOOM. It is selected automatically by `dstrings.h` when the `FRENCH` preprocessor symbol is not defined. The file covers strings used across many subsystems: menus, HUD, automap, status bar, the finale text sequences, and the end-of-game character cast screen.

This file contains no code, no variables, and no type definitions — only `#define` constants. To localise DOOM into another language, the corresponding string in this file (or its French equivalent, `d_french.h`) is changed.

---

## Global Variables

None. This is a header-only file containing preprocessor string constants.

---

## Functions

None defined in this file.

---

## Data Structures

None defined in this file.

---

## String Categories

### D_Main strings
| Macro | Value | Used by |
|-------|-------|---------|
| `D_DEVSTR` | `"Development mode ON.\n"` | `d_main.c` when `-devparm` is set |
| `D_CDROM` | `"CD-ROM Version: default.cfg from c:\\doomdata\n"` | `d_main.c` when `-cdrom` is set |

### M_Menu strings
Menu prompts and confirmations:
- `PRESSKEY`, `PRESSYN` — generic key-press prompts.
- `QUITMSG` — quit confirmation.
- `LOADNET`, `QLOADNET`, `QSAVESPOT`, `SAVEDEAD` — error messages for illegal operations.
- `QSPROMPT`, `QLPROMPT` — quick-save/quick-load confirmation prompts.
- `NEWGAME`, `NIGHTMARE`, `SWSTRING` — new game/episode restrictions.
- `MSGOFF`, `MSGON`, `NETEND`, `ENDGAME` — toggle/end notifications.
- `DOSY` — press-Y-to-quit prompt appended to the quit message.
- `DETAILHI`, `DETAILLO` — detail level labels.
- `GAMMALVL0`–`GAMMALVL4` — gamma correction level labels.
- `EMPTYSTRING` — label for an empty save-game slot.

### P_inter strings
Item pickup notifications displayed in the HUD:
- Armour: `GOTARMOR`, `GOTMEGA`
- Health: `GOTHTHBONUS`, `GOTARMBONUS`, `GOTSTIM`, `GOTMEDINEED`, `GOTMEDIKIT`, `GOTSUPER`
- Keys: `GOTBLUECARD`, `GOTYELWCARD`, `GOTREDCARD`, `GOTBLUESKUL`, `GOTYELWSKUL`, `GOTREDSKULL`
- Powerups: `GOTINVUL`, `GOTBERSERK`, `GOTINVIS`, `GOTSUIT`, `GOTMAP`, `GOTVISOR`, `GOTMSPHERE`
- Ammo: `GOTCLIP`, `GOTCLIPBOX`, `GOTROCKET`, `GOTROCKBOX`, `GOTCELL`, `GOTCELLBOX`, `GOTSHELLS`, `GOTSHELLBOX`, `GOTBACKPACK`
- Weapons: `GOTBFG9000`, `GOTCHAINGUN`, `GOTCHAINSAW`, `GOTLAUNCHER`, `GOTPLASMA`, `GOTSHOTGUN`, `GOTSHOTGUN2`

### P_Doors strings
Key-lock error messages:
- `PD_BLUEO`, `PD_REDO`, `PD_YELLOWO` — object activation key messages.
- `PD_BLUEK`, `PD_REDK`, `PD_YELLOWK` — door-opening key messages.

### G_game string
- `GGSAVED` — "game saved." confirmation.

### HU_stuff strings
HUD and network chat:
- `HUSTR_MSGU` — unsent message notification.
- `HUSTR_E1M1`–`HUSTR_E4M9` — DOOM 1 episode/map names.
- `HUSTR_1`–`HUSTR_32` — DOOM 2 level names.
- `PHUSTR_1`–`PHUSTR_32` — Plutonia map names.
- `THUSTR_1`–`THUSTR_32` — TNT: Evilution map names.
- `HUSTR_CHATMACRO0`–`HUSTR_CHATMACRO9` — default chat macros.
- `HUSTR_TALKTOSELF1`–`HUSTR_TALKTOSELF5` — single-player "talk to self" messages.
- `HUSTR_MESSAGESENT` — sent-message confirmation.
- `HUSTR_PLRGREEN`, `HUSTR_PLRINDIGO`, `HUSTR_PLRBROWN`, `HUSTR_PLRRED` — player-colour prefixes for chat.
- `HUSTR_KEYGREEN`, `HUSTR_KEYINDIGO`, `HUSTR_KEYBROWN`, `HUSTR_KEYRED` — single-character shortcuts for addressing chat to a player.

### AM_map strings
Automap status messages:
- `AMSTR_FOLLOWON`, `AMSTR_FOLLOWOFF` — follow mode toggle.
- `AMSTR_GRIDON`, `AMSTR_GRIDOFF` — grid toggle.
- `AMSTR_MARKEDSPOT` — mark-placed notification.
- `AMSTR_MARKSCLEARED` — all-marks-cleared notification.

### ST_stuff strings
Status bar cheat-code feedback:
- `STSTR_MUS`, `STSTR_NOMUS` — music change.
- `STSTR_DQDON`, `STSTR_DQDOFF` — god mode toggle.
- `STSTR_KFAADDED`, `STSTR_FAADDED` — full ammo cheats.
- `STSTR_NCON`, `STSTR_NCOFF` — no-clip toggle.
- `STSTR_BEHOLD` — power-up cheat prompt.
- `STSTR_BEHOLDX` — power-up toggled confirmation.
- `STSTR_CHOPPERS` — chainsaw cheat message.
- `STSTR_CLEV` — level-change notification.

### F_Finale strings
Multi-line end-of-episode text screens:
- `E1TEXT`–`E4TEXT` — DOOM 1 episode endings.
- `C1TEXT`–`C6TEXT` — DOOM 2 chapter endings.
- `P1TEXT`–`P6TEXT` — Plutonia endings.
- `T1TEXT`–`T6TEXT` — TNT: Evilution endings.

### F_FINALE cast strings
Names displayed during the DOOM 2 cast parade:
- `CC_ZOMBIE`, `CC_SHOTGUN`, `CC_HEAVY`, `CC_IMP`, `CC_DEMON`, `CC_LOST`, `CC_CACO`, `CC_HELL`, `CC_BARON`, `CC_ARACH`, `CC_PAIN`, `CC_REVEN`, `CC_MANCU`, `CC_ARCH`, `CC_SPIDER`, `CC_CYBER`, `CC_HERO`

---

## Dependencies

| Module | Usage |
|--------|-------|
| `dstrings.h` | Includes this file when `FRENCH` is not defined. |

No other direct dependencies. All content is preprocessor text macros.
