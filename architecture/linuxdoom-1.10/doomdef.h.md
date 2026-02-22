# File Overview

`doomdef.h` is the core global configuration and type-definition header for the DOOM engine. It defines the game's version number, all major enumerated types (game mode, game mission, language, game state, skill level, weapons, ammo, keys, power-ups), screen dimension constants, keyboard key codes, and numerous compile-time flags that control optional subsystems (sound server, video mode, debugging).

It is included — directly or transitively — by almost every source file in the engine, making it the "master definitions" file for shared constants and types. The file comment describes it as: "Internally used data structures for virtually everything, key definitions, lots of other stuff."

## Global Variables

`doomdef.h` declares no global variables. All global state is in `doomstat.h` / `doomstat.c`. The `VERSION` constant is defined as an anonymous enum value rather than a variable.

## Functions

No functions are declared in this header.

## Data Structures

### Version

```c
enum { VERSION = 110 };
```

The DOOM version number (1.10). Used in network setup packets to verify version compatibility between nodes, and in savegame headers.

---

### `GameMode_t` (enum)

Identifies which IWAD (game data file) is loaded, controlling which episodes, maps, and animations are available.

| Value | IWAD | Episodes / Maps |
|-------|------|-----------------|
| `shareware` | DOOM 1 shareware | Episode 1 only, 9 maps |
| `registered` | DOOM 1 registered | Episodes 1-3, 27 maps |
| `commercial` | DOOM 2 retail | 1 episode, 34 maps |
| `retail` | DOOM 1 Ultimate | Episodes 1-4, 36 maps |
| `indetermined` | No IWAD found | Undefined behavior |

---

### `GameMission_t` (enum)

Distinguishes between the different official DOOM mission packs. Useful for TC (total conversion) mods and mission-specific logic.

| Value | Product |
|-------|---------|
| `doom` | DOOM 1 |
| `doom2` | DOOM 2 |
| `pack_tnt` | TNT: Evilution |
| `pack_plut` | The Plutonia Experiment |
| `none` | Unspecified |

---

### `Language_t` (enum)

Software localization identifier. Selects the appropriate string tables.

| Value | Language |
|-------|----------|
| `english` | English (default) |
| `french` | French |
| `german` | German |
| `unknown` | Unidentified |

---

### `gamestate_t` (enum)

Represents the high-level state of the engine, controlling which subsystem runs during the main game loop.

| Value | Meaning |
|-------|---------|
| `GS_LEVEL` | In-game play (renders level, runs physics). |
| `GS_INTERMISSION` | Between-level intermission/stats screen. |
| `GS_FINALE` | End-game finale (text crawl, cast sequence). |
| `GS_DEMOSCREEN` | Title screen / demo loop playback. |

---

### `skill_t` (enum)

The five difficulty levels.

| Value | Display Name |
|-------|-------------|
| `sk_baby` | I'm Too Young To Die |
| `sk_easy` | Hey, Not Too Rough |
| `sk_medium` | Hurt Me Plenty |
| `sk_hard` | Ultra-Violence |
| `sk_nightmare` | Nightmare! |

---

### `card_t` (enum)

The six key types in the game. Used to index `player_t::cards[]`.

| Value | Key |
|-------|-----|
| `it_bluecard` | Blue keycard |
| `it_yellowcard` | Yellow keycard |
| `it_redcard` | Red keycard |
| `it_blueskull` | Blue skull key |
| `it_yellowskull` | Yellow skull key |
| `it_redskull` | Red skull key |
| `NUMCARDS` | Count sentinel (= 6) |

---

### `weapontype_t` (enum)

All player weapon types. Also used to signal "no pending weapon change."

| Value | Weapon |
|-------|--------|
| `wp_fist` | Fist / berserk punch |
| `wp_pistol` | Pistol |
| `wp_shotgun` | Shotgun |
| `wp_chaingun` | Chaingun |
| `wp_missile` | Rocket launcher |
| `wp_plasma` | Plasma rifle |
| `wp_bfg` | BFG 9000 |
| `wp_chainsaw` | Chainsaw |
| `wp_supershotgun` | Super shotgun (DOOM 2) |
| `NUMWEAPONS` | Count sentinel (= 9) |
| `wp_nochange` | No pending weapon switch |

---

### `ammotype_t` (enum)

Ammo categories. Used to index `player_t::ammo[]` and `player_t::maxammo[]`.

| Value | Ammo Type | Used By |
|-------|-----------|---------|
| `am_clip` | Bullets | Pistol, Chaingun |
| `am_shell` | Shells | Shotgun, Super Shotgun |
| `am_cell` | Energy cells | Plasma Rifle, BFG |
| `am_misl` | Rockets | Rocket Launcher |
| `NUMAMMO` | Count sentinel (= 4) | |
| `am_noammo` | Unlimited | Chainsaw, Fist |

---

### `powertype_t` (enum)

Power-up types. Used to index `player_t::powers[]`.

| Value | Power-up |
|-------|---------|
| `pw_invulnerability` | God mode (temporary) |
| `pw_strength` | Berserk (fist power + full health) |
| `pw_invisibility` | Partial invisibility (Spectre blur) |
| `pw_ironfeet` | Radiation shielding suit |
| `pw_allmap` | Computer area map (full map reveal) |
| `pw_infrared` | Light amplification visor |
| `NUMPOWERS` | Count sentinel (= 6) |

---

### `powerduration_t` (enum)

Duration in tics for timed power-ups (at 35 tics/second):

| Value | Duration | Seconds |
|-------|----------|---------|
| `INVULNTICS` | `30 * TICRATE` | 30 seconds |
| `INVISTICS` | `60 * TICRATE` | 60 seconds |
| `INFRATICS` | `120 * TICRATE` | 120 seconds |
| `IRONTICS` | `60 * TICRATE` | 60 seconds |

## Compile-Time Constants

### Screen Dimensions

| Constant | Value | Description |
|----------|-------|-------------|
| `BASE_WIDTH` | `320` | Reference screen width in pixels. |
| `SCREEN_MUL` | `1` | Screen scale multiplier (non-functional for dynamic resize). |
| `INV_ASPECT_RATIO` | `0.625` | Inverse aspect ratio (height/width). |
| `SCREENWIDTH` | `320` | Actual screen width used in rendering. |
| `SCREENHEIGHT` | `200` | Actual screen height used in rendering. |
| `MAXPLAYERS` | `4` | Maximum simultaneous players. |
| `TICRATE` | `35` | Game ticks per second. |

### Difficulty/Skill Flags (for `mapthing_t::options`)

| Constant | Value | Meaning |
|----------|-------|---------|
| `MTF_EASY` | `1` | Thing appears on I'm Too Young to Die and Hey, Not Too Rough. |
| `MTF_NORMAL` | `2` | Thing appears on Hurt Me Plenty. |
| `MTF_HARD` | `4` | Thing appears on Ultra-Violence and Nightmare. |
| `MTF_AMBUSH` | `8` | Monster is "deaf" (does not react to sound, only sight). |

### Compile Flags

| Constant | Purpose |
|----------|---------|
| `RANGECHECK` | When defined, enables bounds-checking assertions throughout the renderer. |
| `SNDSERV 1` | Uses the external `sndserver` binary for sound (default). When commented out, enables experimental integrated sound. |
| `SNDINTR` | (Commented out) Experimental asynchronous timer-based sound. |
| `X11_DGA` | (Commented out) Use XFree86 DGA for proper mouse sampling instead of MIT SHM. |

### Keyboard Key Codes

These are defined as hex values matching DOS scan codes offset by `0x80` for extended keys:

| Constant | Value | Key |
|----------|-------|-----|
| `KEY_RIGHTARROW` | `0xae` | Right arrow |
| `KEY_LEFTARROW` | `0xac` | Left arrow |
| `KEY_UPARROW` | `0xad` | Up arrow |
| `KEY_DOWNARROW` | `0xaf` | Down arrow |
| `KEY_ESCAPE` | `27` | Escape |
| `KEY_ENTER` | `13` | Enter |
| `KEY_TAB` | `9` | Tab |
| `KEY_F1`–`KEY_F10` | `0x80+0x3b`–`0x80+0x44` | Function keys F1-F10 |
| `KEY_F11` | `0x80+0x57` | F11 |
| `KEY_F12` | `0x80+0x58` | F12 |
| `KEY_BACKSPACE` | `127` | Backspace |
| `KEY_PAUSE` | `0xff` | Pause |
| `KEY_RSHIFT` | `0x80+0x36` | Right Shift |
| `KEY_RCTRL` | `0x80+0x1d` | Right Ctrl |
| `KEY_RALT` | `0x80+0x38` | Right Alt |
| `KEY_LALT` | `KEY_RALT` | Left Alt (same code) |

## Dependencies

| File | Reason |
|------|--------|
| `<stdio.h>` | `FILE*` type (used by `debugfile` declared in `doomstat.h`). |
| `<string.h>` | String functions used throughout the engine. |

Many other headers are referenced but commented out in this file; they were previously included here but extracted to reduce coupling between modules.
