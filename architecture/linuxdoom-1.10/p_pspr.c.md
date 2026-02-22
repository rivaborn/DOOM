# File Overview

**Source file:** `linuxdoom-1.10/p_pspr.c`

`p_pspr.c` manages **player sprites** — the weapon animations drawn in the foreground of the player's view. It implements:

1. The **weapon state machine**: advancing psprite animation frames each tic and calling weapon action callbacks.
2. The **weapon lifecycle**: bringing a weapon up from the bottom of the screen, lowering it when switching, and displaying it in the ready/idle position.
3. All **weapon attack action functions** (`A_Fire*`, `A_Punch`, `A_Saw`, `A_BFGSpray`) that are called as state-machine callbacks when the player fires.
4. **Support routines** shared across multiple weapons: `P_BulletSlope` for auto-aim, `P_GunShot` for single-pellet hitscan damage.
5. **Weapon management utilities**: `P_CheckAmmo` (checks ammo and selects a fallback weapon), `P_FireWeapon`, `P_DropWeapon`, `P_SetupPsprites`, `P_MovePsprites`.

The weapon system uses two overlaid sprites per player, called **psprites** (player sprites): `ps_weapon` (the main weapon body) and `ps_flash` (the muzzle flash overlay). Each has its own independent state and screen-space position.

---

## Architecture: The Weapon State Machine

Every weapon is described by a set of state indices in `weaponinfo[]` (from `d_items.c`/`d_items.h`):

| Field | Meaning |
|-------|---------|
| `upstate` | State entered when the weapon is being raised |
| `downstate` | State entered when the weapon is being lowered |
| `readystate` | State entered when the weapon is fully raised and idle |
| `atkstate` | State entered when the fire button is pressed |
| `flashstate` | State for the muzzle-flash psprite overlay |
| `ammo` | Ammo type consumed |

Each `state_t` has an `action.acp2` function pointer that takes `(player_t*, pspdef_t*)`. These action callbacks implement the actual shooting behaviour. The machine loops: ready -> attack -> flash -> ready.

`P_SetPsprite` is the FSM driver. `P_MovePsprites` is called once per tic by the player-thinking code (`p_user.c`) to decrement tic counts and advance states.

---

## Global Variables

| Type | Name | Purpose |
|------|------|---------|
| `fixed_t` | `swingx` | Computed weapon swing offset in X, updated by `P_CalcSwing`. Not currently used in the main render path (the code that calls `P_CalcSwing` is present but the swing values are read only if explicitly wired in). |
| `fixed_t` | `swingy` | Computed weapon swing offset in Y. Same status as `swingx`. |
| `fixed_t` | `bulletslope` | Auto-aim slope result, set by `P_BulletSlope`. Shared between `P_BulletSlope` and all hitscan weapon action functions. Represents the vertical firing angle as a fixed-point slope. |

---

## Preprocessor Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `LOWERSPEED` | `FRACUNIT*6` | Pixels per tic that a weapon descends when being lowered. |
| `RAISESPEED` | `FRACUNIT*6` | Pixels per tic that a weapon rises when being raised. |
| `WEAPONBOTTOM` | `128*FRACUNIT` | The Y screen coordinate (in fixed-point) when the weapon is fully off-screen at the bottom. Used as the starting position when bringing up a weapon. |
| `WEAPONTOP` | `32*FRACUNIT` | The Y screen coordinate when the weapon is fully raised and at rest. |
| `BFGCELLS` | `40` | Number of plasma cells consumed per BFG shot. |

---

## Functions

### `P_SetPsprite`

```c
void P_SetPsprite(player_t *player, int position, statenum_t stnum)
```

**Purpose:** Drives the psprite finite-state machine for one weapon slot. Advances through zero-tic states immediately and fires action callbacks as each new state is entered. This is the psprite equivalent of `P_SetMobjState`.

**Parameters:**
- `player` - the player whose psprite is being updated.
- `position` - which psprite slot: `ps_weapon` (0) or `ps_flash` (1), from `psprnum_t`.
- `stnum` - target state number. A value of `S_NULL` (0) deactivates the psprite (`psp->state = NULL`).

**Key logic:**
1. If `stnum == 0`, null the state and stop.
2. Look up `states[stnum]`, copy `tics` into `psp->tics`.
3. If the state has `misc1` set, update `psp->sx` and `psp->sy` (screen position override in fixed-point).
4. If the state has an `action.acp2` callback, call `state->action.acp2(player, psp)`. After the call, check if the state was nulled (the action may have removed the psprite).
5. Read `psp->state->nextstate` and loop if `psp->tics == 0` (zero-tic transitional state).

---

### `P_CalcSwing`

```c
void P_CalcSwing(player_t *player)
```

**Purpose:** Calculates a sinusoidal weapon-swing offset based on the player's bob magnitude and the current level time. The result is stored in the module-level `swingx` and `swingy` globals. This function is defined but the call sites that would use the computed values are commented out or unused; the feature was apparently planned but not completed.

**Parameters:**
- `player` - used to read `player->bob` (the movement-speed-derived bob amplitude).

**Key logic:**
- X swing: `swingx = bob * sin(FINEANGLES/70 * leveltime)`.
- Y swing: `swingy = -swingx * sin(FINEANGLES/70 * leveltime + FINEANGLES/2)`.

---

### `P_BringUpWeapon`

```c
void P_BringUpWeapon(player_t *player)
```

**Purpose:** Starts the weapon-raise animation sequence for the pending weapon. Plays the chainsaw startup sound if the pending weapon is the chainsaw. Sets the weapon psprite Y position to `WEAPONBOTTOM` (off-screen) and transitions to the weapon's `upstate`.

**Parameters:**
- `player` - the player whose pending weapon is to be raised.

**Key logic:**
- If `pendingweapon == wp_nochange`, falls back to `readyweapon` (no actual change pending).
- Clears `pendingweapon = wp_nochange` after reading it.
- Calls `P_SetPsprite(player, ps_weapon, weaponinfo[pendingweapon].upstate)`.

---

### `P_CheckAmmo`

```c
boolean P_CheckAmmo(player_t *player)
```

**Purpose:** Checks whether the player has enough ammo to fire the current weapon. If not, selects the best available fallback weapon (by priority order) and initiates the weapon-lower animation.

**Parameters:**
- `player` - the player to check.

**Returns:** `true` if sufficient ammo is available; `false` if the weapon needs to switch.

**Priority order for fallback weapon selection:**
1. Plasma gun (if owned, has cells, and not shareware mode)
2. Super shotgun (if owned, has 2+ shells, and commercial/Doom 2 mode)
3. Chaingun (if owned, has clips)
4. Shotgun (if owned, has shells)
5. Pistol (if has clips)
6. Chainsaw (if owned)
7. Rocket launcher (if owned, has missiles)
8. BFG (if owned, has 40+ cells, and not shareware)
9. Fist (always available as last resort)

**Ammo thresholds:**
- BFG requires `BFGCELLS` (40) cells.
- Super shotgun requires 2 shells.
- All other weapons require 1 unit.

---

### `P_FireWeapon`

```c
void P_FireWeapon(player_t *player)
```

**Purpose:** Initiates a weapon firing sequence. Checks ammo, sets the player mobj to the attack animation state, transitions the weapon psprite to the attack state, and calls `P_NoiseAlert` to wake nearby monsters.

**Parameters:**
- `player` - the firing player.

**Key logic:**
- Returns immediately if `P_CheckAmmo` returns false (no ammo, initiates weapon switch instead).
- `P_SetMobjState(player->mo, S_PLAY_ATK1)` sets the player's body to the attack pose.
- Weapon-specific attack logic is handled by `A_Fire*` callbacks in the attack state.

---

### `P_DropWeapon`

```c
void P_DropWeapon(player_t *player)
```

**Purpose:** Initiates weapon lowering on player death. Transitions the weapon psprite to the current weapon's `downstate`.

**Parameters:**
- `player` - the dying player.

---

### `A_WeaponReady`

```c
void A_WeaponReady(player_t *player, pspdef_t *psp)
```

**Purpose:** State-machine action called every tic while the weapon is in the ready/idle state. Handles the full set of interactions: escaping attack pose, playing the chainsaw idle sound, initiating a weapon switch, firing, and applying the weapon bob.

**Parameters:**
- `player` - the owning player.
- `psp` - the weapon psprite.

**Key logic:**
1. If player mobj is in `S_PLAY_ATK1` or `S_PLAY_ATK2`, revert to `S_PLAY` (escape attack pose).
2. If the chainsaw is ready and in `S_SAW`, play `sfx_sawidl`.
3. If `pendingweapon != wp_nochange` or the player is dead (`health == 0`), lower the weapon.
4. If the fire button is held:
   - For the rocket launcher and BFG, auto-fire is suppressed (`attackdown` is checked). Other weapons allow holding the fire button.
   - Sets `player->attackdown = true` and calls `P_FireWeapon`.
5. Clears `attackdown` when the fire button is released.
6. Applies weapon bob: `psp->sx = FRACUNIT + bob*cos(angle)` and `psp->sy = WEAPONTOP + bob*sin(angle)` where angle increments at `128*leveltime`.

---

### `A_ReFire`

```c
void A_ReFire(player_t *player, pspdef_t *psp)
```

**Purpose:** Called at the end of a weapon's fire animation sequence to check whether the player is still holding the fire button for a rapid-fire followup. Increments the `refire` counter used by some weapons to determine spread accuracy.

**Key logic:**
- If the fire button is still held, no weapon change is pending, and the player is alive: increments `player->refire` and calls `P_FireWeapon` again.
- Otherwise: resets `player->refire = 0` and calls `P_CheckAmmo` (which will initiate a weapon switch if needed).

---

### `A_CheckReload`

```c
void A_CheckReload(player_t *player, pspdef_t *psp)
```

**Purpose:** Calls `P_CheckAmmo` to trigger a weapon switch if ammo runs out mid-animation. Intended as a mid-sequence check for the super shotgun so it does not complete its reload animation when empty. The conditional code (`#if 0`) for a dedicated "no reload" animation was disabled.

---

### `A_Lower`

```c
void A_Lower(player_t *player, pspdef_t *psp)
```

**Purpose:** Called each tic during the weapon-lower animation. Moves the weapon sprite down by `LOWERSPEED` pixels. When the sprite reaches `WEAPONBOTTOM` (fully off-screen), commits the weapon switch by setting `readyweapon = pendingweapon` and calling `P_BringUpWeapon`. Handles the dead-player case by keeping the weapon off-screen permanently.

**Key logic:**
- `psp->sy += LOWERSPEED` each tic.
- While `psp->sy < WEAPONBOTTOM`, return (not at bottom yet).
- If `playerstate == PST_DEAD`, clamp `sy = WEAPONBOTTOM` and return without switching.
- If `health == 0` (just died), set the psprite to `S_NULL` (vanish completely).
- Otherwise, set `readyweapon = pendingweapon` and call `P_BringUpWeapon`.

---

### `A_Raise`

```c
void A_Raise(player_t *player, pspdef_t *psp)
```

**Purpose:** Called each tic during the weapon-raise animation. Moves the weapon sprite up by `RAISESPEED` pixels. When it reaches `WEAPONTOP` (fully raised), transitions to the weapon's `readystate`.

**Key logic:**
- `psp->sy -= RAISESPEED` each tic.
- While `psp->sy > WEAPONTOP`, return (not at top yet).
- Clamp to `WEAPONTOP` and transition via `P_SetPsprite(player, ps_weapon, weaponinfo[readyweapon].readystate)`.

---

### `A_GunFlash`

```c
void A_GunFlash(player_t *player, pspdef_t *psp)
```

**Purpose:** Sets the player body to the second attack pose (`S_PLAY_ATK2`) and activates the flash psprite slot with the weapon's muzzle-flash animation. Called from within attack state sequences that require a separate flash layer.

---

### `A_Punch`

```c
void A_Punch(player_t *player, pspdef_t *psp)
```

**Purpose:** Performs the fist melee attack. Deals 2-20 damage (random), multiplied by 10 if the berserk powerup (`pw_strength`) is active. Applies a small random angular spread. Turns the player to face the hit target.

**Key logic:**
- `damage = (P_Random()%10 + 1) * 2`, times 10 with berserk.
- Angular jitter: `angle += (P_Random()-P_Random()) << 18`.
- Uses `P_AimLineAttack` + `P_LineAttack` at `MELEERANGE`.
- If `linetarget` is set: plays `sfx_punch` and rotates player to face target.

---

### `A_Saw`

```c
void A_Saw(player_t *player, pspdef_t *psp)
```

**Purpose:** Performs the chainsaw melee attack. Similar to `A_Punch` but with a gradualy-tracking turning mechanic: if the player's angle and the target angle differ by more than `ANG90/20`, the player snaps to within `ANG90/21`; otherwise gently adjusts by `ANG90/20`. Plays distinct sounds for hit and miss.

**Key logic:**
- Damage: `2 * (P_Random()%10 + 1)`.
- Range: `MELEERANGE + 1` to ensure the puff's flash state is visible.
- On miss: plays `sfx_sawful`. On hit: plays `sfx_sawhit`, then adjusts player angle and sets `MF_JUSTATTACKED`.

---

### `A_FireMissile`

```c
void A_FireMissile(player_t *player, pspdef_t *psp)
```

**Purpose:** Fires a rocket from the rocket launcher. Decrements `am_misl` ammo by 1 and calls `P_SpawnPlayerMissile(player->mo, MT_ROCKET)`.

---

### `A_FireBFG`

```c
void A_FireBFG(player_t *player, pspdef_t *psp)
```

**Purpose:** Fires the BFG 9000 projectile. Decrements `am_cell` ammo by `BFGCELLS` (40) and calls `P_SpawnPlayerMissile(player->mo, MT_BFG)`. The actual area-damage sweep is handled by `A_BFGSpray` which is triggered when the BFG projectile reaches its target.

---

### `A_FirePlasma`

```c
void A_FirePlasma(player_t *player, pspdef_t *psp)
```

**Purpose:** Fires one plasma ball from the plasma gun. Decrements cells by 1. Randomises the flash state between two frames (`flashstate + (P_Random()&1)`) for visual variety. Calls `P_SpawnPlayerMissile(player->mo, MT_PLASMA)`.

---

### `P_BulletSlope`

```c
void P_BulletSlope(mobj_t *mo)
```

**Purpose:** Performs the three-pass auto-aim sweep used by all hitscan weapons, storing the resulting vertical slope in the module-level `bulletslope` global. This is the same algorithm as `P_SpawnPlayerMissile`'s aim logic.

**Key logic:**
- Pass 1: aim straight ahead (`mo->angle`).
- Pass 2: aim slightly left (`+1<<26`) if pass 1 found no target.
- Pass 3: aim slightly right (`-2<<26` from the left-shifted angle) if still no target.
- If no target is found after three passes, `bulletslope` is whatever the last `P_AimLineAttack` returned (typically 0).

---

### `P_GunShot`

```c
void P_GunShot(mobj_t *mo, boolean accurate)
```

**Purpose:** Fires a single hitscan bullet pellet using the pre-computed `bulletslope`. Used by pistol and chaingun. Damage is 5-15 (random in increments of 5). If `accurate` is false, adds a random horizontal spread of up to `±(P_Random()-P_Random())<<18`.

**Parameters:**
- `mo` - the firing mobj (the player's mobj).
- `accurate` - true for the first shot in a burst (e.g., first pistol shot); false for refiring or shotgun pellets.

---

### `A_FirePistol`

```c
void A_FirePistol(player_t *player, pspdef_t *psp)
```

**Purpose:** Fires the pistol. Plays `sfx_pistol`, sets the player body to attack pose 2, decrements clip ammo by 1, activates the flash psprite, calls `P_BulletSlope` for auto-aim, then fires one pellet via `P_GunShot`. The first shot of a burst is accurate (`!player->refire`); subsequent refires spread.

---

### `A_FireShotgun`

```c
void A_FireShotgun(player_t *player, pspdef_t *psp)
```

**Purpose:** Fires the standard shotgun. Plays `sfx_shotgn`, sets attack pose 2, decrements shells by 1, activates flash, computes bullet slope, then fires **7 inaccurate pellets** via a loop of `P_GunShot(player->mo, false)`.

---

### `A_FireShotgun2`

```c
void A_FireShotgun2(player_t *player, pspdef_t *psp)
```

**Purpose:** Fires the super shotgun (Doom 2). Plays `sfx_dshtgn`, decrements shells by **2**, fires **20 pellets** each with independent random angle and slope offsets. Angle spread: `(P_Random()-P_Random()) << 19`. Slope spread: `bulletslope + (P_Random()-P_Random()) << 5`.

---

### `A_FireCGun`

```c
void A_FireCGun(player_t *player, pspdef_t *psp)
```

**Purpose:** Fires one bullet from the chaingun. The flash state is offset by the current animation frame relative to `S_CHAIN1`, so the two chaingun states produce alternating flash frames. Returns early (without firing) if ammo is already zero when the callback fires (safety check).

---

### `A_Light0`, `A_Light1`, `A_Light2`

```c
void A_Light0(player_t *player, pspdef_t *psp)
void A_Light1(player_t *player, pspdef_t *psp)
void A_Light2(player_t *player, pspdef_t *psp)
```

**Purpose:** Set the player's `extralight` level (0, 1, or 2 additional light sectors). Used in weapon flash states to briefly illuminate the surroundings when a weapon fires. `A_Light0` is used in the ready state to return to normal lighting; `A_Light1`/`A_Light2` are used in flash states to create the muzzle-flash light burst.

---

### `A_BFGSpray`

```c
void A_BFGSpray(mobj_t *mo)
```

**Purpose:** The BFG9000 area-damage callback. Called from the BFG projectile's impact state. Sweeps 40 rays in a 90-degree arc centred on the BFG's current angle, finds any monster in line-of-sight, spawns a `MT_EXTRABFG` effect on it, and deals damage.

**Parameters:**
- `mo` - the BFG projectile mobj. `mo->target` is the player who fired it.

**Key logic:**
- 40 rays spaced `ANG90/40` apart, covering `[-ANG90/2, +ANG90/2]` relative to `mo->angle`.
- `P_AimLineAttack(mo->target, an, 16*64*FRACUNIT)` — aims from the **player**, not the projectile, at each ray angle.
- For each `linetarget` found: spawns `MT_EXTRABFG` at the target's position, then deals `sum(15 * (P_Random()&7 + 1))` damage (average ~60 per target, up to 120).
- Total BFG damage to a single target = direct hit (if applicable) + spray damage.

---

### `A_BFGsound`

```c
void A_BFGsound(player_t *player, pspdef_t *psp)
```

**Purpose:** Plays `sfx_bfg` when the BFG fires. Called from the weapon's attack state before the projectile is spawned.

---

### `P_SetupPsprites`

```c
void P_SetupPsprites(player_t *player)
```

**Purpose:** Initialises the psprite system for a player at the start of a level or after a respawn. Clears all psprite states to NULL, then immediately begins bringing up the player's current weapon.

**Key logic:**
- Loops over all `NUMPSPRITES` slots, setting `state = NULL`.
- Sets `pendingweapon = readyweapon` and calls `P_BringUpWeapon`.

---

### `P_MovePsprites`

```c
void P_MovePsprites(player_t *player)
```

**Purpose:** Called once per tic by the player-thinking routine. Advances the tic counters for both psprite slots and triggers state transitions when a tic count expires. Also synchronises the flash psprite's screen position to the weapon psprite.

**Key logic:**
1. For each of the two psprite slots (`ps_weapon`, `ps_flash`):
   - If `psp->state` is non-null, decrement `psp->tics`.
   - If `tics == 0`, call `P_SetPsprite(player, i, psp->state->nextstate)` to advance.
   - A `tics` value of `-1` is permanent and never advances.
2. Copies `ps_weapon`'s `sx` and `sy` to `ps_flash` so the flash overlay tracks the weapon position.

---

## Data Structures

`p_pspr.c` uses structures defined in `p_pspr.h` and `d_player.h`. Key types:

### `pspdef_t` (from `p_pspr.h`)
Per-slot psprite state:
- `state_t* state` — current animation state (NULL = slot inactive).
- `int tics` — tics remaining in current state.
- `fixed_t sx, sy` — screen-space position in fixed-point (320x200 coordinate space).

### `psprnum_t` (from `p_pspr.h`)
Enum: `ps_weapon` (0), `ps_flash` (1), `NUMPSPRITES` (2).

---

## Dependencies

| Header / Module | What is used |
|----------------|--------------|
| `doomdef.h` | Game constants, `boolean`, `FRACUNIT`, angle constants |
| `d_event.h` | `BT_ATTACK` button bit |
| `m_random.h` | `P_Random` for damage/spread/animation jitter |
| `p_local.h` | `P_AimLineAttack`, `P_LineAttack`, `P_NoiseAlert`, `P_SpawnMobj`, `P_SpawnPlayerMissile`, `P_DamageMobj`, `MELEERANGE`, `MISSILERANGE` |
| `s_sound.h` | `S_StartSound` |
| `doomstat.h` | `leveltime`, `gamemode`, `linetarget` |
| `sounds.h` | Sound effect IDs (`sfx_pistol`, `sfx_shotgn`, `sfx_dshtgn`, `sfx_punch`, `sfx_sawidl`, `sfx_sawup`, `sfx_sawful`, `sfx_sawhit`, `sfx_bfg`) |
| `p_pspr.h` | `pspdef_t`, `psprnum_t`, `FF_FULLBRIGHT`, `FF_FRAMEMASK` |
| `info.h` | `states[]`, `weaponinfo[]`, `state_t`, `statenum_t` |
| `d_player.h` | `player_t`, `pspdef_t`, weapon and power-up enumerations |
| `tables.h` | `finecosine[]`, `finesine[]`, `FINEMASK`, `FINEANGLES`, `ANGLETOFINESHIFT` |
| `r_main.h` | `R_PointToAngle2` (used in `A_Punch`, `A_Saw`) |
