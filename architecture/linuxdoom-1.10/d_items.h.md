# File Overview

`d_items.h` declares the `weaponinfo_t` struct and the `weaponinfo[]` extern array that together describe the animation states and ammunition requirements for every weapon in the game. This header is the public interface between `d_items.c` (which initialises the table) and the player sprite (`p_pspr.c`) and status bar code that reads it at runtime.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `weaponinfo_t[NUMWEAPONS]` | `weaponinfo` | Array of weapon descriptors, one per `weapontype_t` value. Defined in `d_items.c`. |

---

## Functions

None defined in this file.

---

## Data Structures

### `weaponinfo_t`
```c
typedef struct {
    ammotype_t  ammo;
    int         upstate;
    int         downstate;
    int         readystate;
    int         atkstate;
    int         flashstate;
} weaponinfo_t;
```
Describes the animation and resource properties of a single weapon:

| Field | Type | Description |
|-------|------|-------------|
| `ammo` | `ammotype_t` | Which ammo pool this weapon draws from. `am_noammo` for melee/chainsaw. |
| `upstate` | `int` | State index for the "raising the weapon" animation. |
| `downstate` | `int` | State index for the "lowering the weapon" animation. |
| `readystate` | `int` | State index for the weapon idle/ready loop. |
| `atkstate` | `int` | State index for the fire/attack animation. |
| `flashstate` | `int` | State index for the muzzle flash overlay. `S_NULL` for weapons with no flash. |

The `int` state fields are indices into the global `states[]` array from `info.c`. They are used by `p_pspr.c` to drive the player-sprite (weapon overlay) state machine.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `doomdef.h` | `ammotype_t`, `weapontype_t`, `NUMWEAPONS` enum values. |
