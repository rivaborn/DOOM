# File Overview

`d_items.c` contains the single global data table that describes every weapon in DOOM: `weaponinfo[]`. For each of the nine weapons (fist, pistol, shotgun, chaingun, rocket launcher, plasma rifle, BFG 9000, chainsaw, super shotgun) the table records which ammunition type the weapon uses and which state-machine states govern its raise, lower, idle, fire, and muzzle-flash animations. The states are integer indices into the global `states[]` array defined in `info.c`.

This file has no logic — it is purely a data initialiser for a struct array declared in `d_items.h`.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `weaponinfo_t[NUMWEAPONS]` | `weaponinfo` | Array of 9 entries (indexed by `weapontype_t`) describing each weapon's ammo type and animation state indices. |

### Table contents

| Index | Weapon | `ammo` | `upstate` | `downstate` | `readystate` | `atkstate` | `flashstate` |
|-------|--------|--------|-----------|-------------|--------------|------------|--------------|
| 0 | Fist | `am_noammo` | `S_PUNCHUP` | `S_PUNCHDOWN` | `S_PUNCH` | `S_PUNCH1` | `S_NULL` |
| 1 | Pistol | `am_clip` | `S_PISTOLUP` | `S_PISTOLDOWN` | `S_PISTOL` | `S_PISTOL1` | `S_PISTOLFLASH` |
| 2 | Shotgun | `am_shell` | `S_SGUNUP` | `S_SGUNDOWN` | `S_SGUN` | `S_SGUN1` | `S_SGUNFLASH1` |
| 3 | Chaingun | `am_clip` | `S_CHAINUP` | `S_CHAINDOWN` | `S_CHAIN` | `S_CHAIN1` | `S_CHAINFLASH1` |
| 4 | Rocket Launcher | `am_misl` | `S_MISSILEUP` | `S_MISSILEDOWN` | `S_MISSILE` | `S_MISSILE1` | `S_MISSILEFLASH1` |
| 5 | Plasma Rifle | `am_cell` | `S_PLASMAUP` | `S_PLASMADOWN` | `S_PLASMA` | `S_PLASMA1` | `S_PLASMAFLASH1` |
| 6 | BFG 9000 | `am_cell` | `S_BFGUP` | `S_BFGDOWN` | `S_BFG` | `S_BFG1` | `S_BFGFLASH1` |
| 7 | Chainsaw | `am_noammo` | `S_SAWUP` | `S_SAWDOWN` | `S_SAW` | `S_SAW1` | `S_NULL` |
| 8 | Super Shotgun | `am_shell` | `S_DSGUNUP` | `S_DSGUNDOWN` | `S_DSGUN` | `S_DSGUN1` | `S_DSGUNFLASH1` |

---

## Functions

None. This file contains only the global variable initialiser.

---

## Data Structures

Defined in `d_items.h` (see that file's documentation); used here by the `weaponinfo[]` initialiser.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `info.h` | State name constants (`S_PUNCHUP`, `S_PISTOL1`, etc.) and sprite enumerations referenced in the table. |
| `d_items.h` | Declaration of `weaponinfo_t` and `weaponinfo[]`. |
