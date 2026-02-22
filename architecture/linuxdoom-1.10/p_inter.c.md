# File Overview

`p_inter.c` is the interaction handler for the DOOM engine, responsible for all gameplay interactions that occur when entities collide or contact each other. This file covers three broad areas: item pickup logic (giving ammo, weapons, health, armor, powerups, and keys to the player), the death processing of map objects (monsters and players), and the damage application system that handles hits from weapons, explosions, and environmental hazards.

The file sits at the center of the play simulation layer (the `p_` prefix denotes "play"). It is called from the map movement code (`p_map.c`) when a player touches a special item, and from weapon code and monster AI when attacks connect. It reads from global tables such as `weaponinfo[]` and `mobjinfo[]` to resolve behavior, and writes into player and mobj structures to record results.

---

## Global Variables

| Type | Name | Purpose |
|------|------|---------|
| `int[NUMAMMO]` | `maxammo` | Maximum ammo capacity per ammo type when no backpack is carried: `{200, 50, 300, 50}` for clip, shell, cell, missile respectively |
| `int[NUMAMMO]` | `clipammo` | Ammo added per "clip load" for each type: `{10, 4, 20, 1}`. This is the base unit used when computing pickups |

The macro `BONUSADD` (value `6`) is defined locally and used to increment `player->bonuscount` whenever an item is picked up, which drives the screen palette flash effect.

---

## Functions

### `P_GiveAmmo`

```c
boolean P_GiveAmmo(player_t* player, ammotype_t ammo, int num)
```

**Purpose:** Adds ammo of the specified type to the player's inventory, respecting the per-type maximum. Automatically switches to a better weapon if the player was completely out.

**Parameters:**
- `player` - The player receiving ammo.
- `ammo` - The ammo type index (`am_clip`, `am_shell`, `am_cell`, `am_misl`). `am_noammo` causes immediate return of `false`.
- `num` - Number of clip loads to give. If `0`, gives half a clip load.

**Return value:** `false` if the ammo could not be picked up at all (wrong type or player already at maximum); `true` if ammo was actually added.

**Key logic:**
1. Validates the ammo type; returns `false` for `am_noammo` or an already-full inventory.
2. Converts clip loads to individual units: `num *= clipammo[ammo]`, or half a clip if `num` is zero.
3. On Baby skill or Nightmare skill, the amount is doubled (`num <<= 1`).
4. Caps the result at `player->maxammo[ammo]`.
5. If `oldammo` was zero (player was completely dry), performs an automatic weapon upgrade based on the ammo type received (e.g., clips trigger chaingun or pistol selection, shells trigger shotgun).

---

### `P_GiveWeapon`

```c
boolean P_GiveWeapon(player_t* player, weapontype_t weapon, boolean dropped)
```

**Purpose:** Grants a weapon to the player along with the appropriate amount of ammo. Handles both network-game and single-player rules.

**Parameters:**
- `player` - The player receiving the weapon.
- `weapon` - Which weapon to give.
- `dropped` - `true` if the weapon was dropped by a dying enemy (gives 1 clip load instead of 2).

**Return value:** `true` if something of value was actually received (new weapon or new ammo); `false` otherwise.

**Key logic:**
- In network games (not deathmatch-2): weapons persist on the map; the player just gets their ammo and the weapon is marked owned. In deathmatch, 5 clip loads are awarded; in cooperative, 2.
- In single player: drops give 1 clip load, placed weapons give 2.
- Sets `player->pendingweapon` to trigger the weapon-raise animation.
- Plays `sfx_wpnup` for the console player.

---

### `P_GiveBody`

```c
boolean P_GiveBody(player_t* player, int num)
```

**Purpose:** Adds health points to the player, clamped at `MAXHEALTH` (100).

**Parameters:**
- `player` - The player to heal.
- `num` - The number of hit points to add.

**Return value:** `false` if the player was already at maximum health; `true` if health was actually added.

**Key logic:** Both `player->health` and `player->mo->health` are updated together to keep them in sync.

---

### `P_GiveArmor`

```c
boolean P_GiveArmor(player_t* player, int armortype)
```

**Purpose:** Grants armor to the player, but only if the new armor is better than what the player already has.

**Parameters:**
- `player` - The player to equip.
- `armortype` - Armor quality: `1` for green armor (100 points), `2` for blue/mega armor (200 points).

**Return value:** `false` if the player's current armor is already equal to or better than the offered armor; `true` if the armor was accepted.

**Key logic:** The armor point threshold for type 1 is 100 (`1*100`) and for type 2 is 200 (`2*100`). A player with 150 armor points from type 2 will refuse a type 1 pickup since 150 >= 100.

---

### `P_GiveCard`

```c
void P_GiveCard(player_t* player, card_t card)
```

**Purpose:** Gives the player a key card or skull key. No-ops if already possessed.

**Parameters:**
- `player` - The player receiving the key.
- `card` - The specific card type (blue/yellow/red card or skull).

**Key logic:** Sets `player->bonuscount` to `BONUSADD` to trigger the palette bonus flash. Cards are tracked as simple integer flags in `player->cards[]`.

---

### `P_GivePower`

```c
boolean P_GivePower(player_t* player, int power)
```

**Purpose:** Activates a powerup for the player by setting its countdown timer.

**Parameters:**
- `player` - The player receiving the power.
- `power` - The powerup type (one of the `powertype_t` enum values).

**Return value:** `true` if the powerup was applied; `false` if the player already has this powerup (for non-timed ones).

**Key logic:**
- `pw_invulnerability`: sets timer to `INVULNTICS`.
- `pw_invisibility`: sets timer to `INVISTICS`, also sets `MF_SHADOW` flag on the player mobj for fuzzy rendering.
- `pw_infrared`: sets timer to `INFRATICS`.
- `pw_ironfeet` (radiation suit): sets timer to `IRONTICS`.
- `pw_strength` (berserk): calls `P_GiveBody(player, 100)` to restore full health, then sets power to 1 (permanent for the level).
- All other powers: returns `false` if already held; sets to `1` otherwise.

---

### `P_TouchSpecialThing`

```c
void P_TouchSpecialThing(mobj_t* special, mobj_t* toucher)
```

**Purpose:** Main dispatch function for item pickup. Called when a player-controlled mobj overlaps a special item mobj (one with `MF_SPECIAL` set). Identifies the item by sprite number and calls the appropriate give function.

**Parameters:**
- `special` - The item being touched.
- `toucher` - The entity doing the touching (must be a player).

**Key logic:**
1. Checks that the item is within vertical reach (`-8` to `toucher->height` units of delta-z).
2. Checks that the toucher is alive.
3. Dispatches on `special->sprite` to identify item type and calls the relevant `P_Give*` function. Returns early (without removing the item) if the give function indicates the item was not needed.
4. Handles network-game cards: in multiplayer, keys are never removed from the map (they return without calling `P_RemoveMobj`).
5. The backpack (`SPR_BPAK`) doubles all `player->maxammo[]` entries (once only) then gives 1 clip load of each type.
6. At the end: increments `player->itemcount` if `MF_COUNTITEM`, calls `P_RemoveMobj(special)`, adds `BONUSADD` to `player->bonuscount`, plays pickup sound for the console player.

**Sprite cases handled:**
- `SPR_ARM1/ARM2` - green/blue armor
- `SPR_BON1/BON2` - health/armor bonuses (can exceed 100%, up to 200%)
- `SPR_SOUL` - soulsphere (+100 HP, cap 200)
- `SPR_MEGA` - megasphere (200 HP + blue armor, Doom 2 only)
- `SPR_BKEY/YKEY/RKEY/BSKU/YSKU/RSKU` - six key types
- `SPR_STIM/MEDI` - stimpak (+10) and medikit (+25)
- `SPR_PINV/PSTR/PINS/SUIT/PMAP/PVIS` - powerups
- `SPR_CLIP/AMMO/ROCK/BROK/CELL/CELP/SHEL/SBOX/BPAK` - ammo and backpack
- `SPR_BFUG/MGUN/CSAW/LAUN/PLAS/SHOT/SGN2` - all weapons

---

### `P_KillMobj`

```c
void P_KillMobj(mobj_t* source, mobj_t* target)
```

**Purpose:** Handles the death of a map object. Transitions the mobj to its death or extreme-death state, updates kill/frag counters, drops loot, and forces player-specific death handling.

**Parameters:**
- `source` - The entity responsible for the kill (may be `NULL` for environmental damage).
- `target` - The entity being killed.

**Key logic:**
1. Clears `MF_SHOOTABLE`, `MF_FLOAT`, `MF_SKULLFLY`; sets `MF_CORPSE | MF_DROPOFF`; halves the height (`>>= 2`) so corpses lie flat.
2. Removes `MF_NOGRAVITY` from non-skull targets so they fall.
3. Increments `source->player->killcount` if the target has `MF_COUNTKILL`. In single player, even monster-on-monster kills count for player 0.
4. Records frags in `source->player->frags[]` for multiplayer.
5. For player targets: clears `MF_SOLID`, sets `PST_DEAD`, calls `P_DropWeapon`, stops the automap if it was active.
6. Chooses extreme death state (`xdeathstate`) if `target->health < -spawnhealth`.
7. Randomizes death animation duration: `target->tics -= P_Random()&3`.
8. Drops loot: `MT_POSSESSED` and `MT_WOLFSS` drop `MT_CLIP`; `MT_SHOTGUY` drops `MT_SHOTGUN`; `MT_CHAINGUY` drops `MT_CHAINGUN`. Dropped items get `MF_DROPPED` set.

---

### `P_DamageMobj`

```c
void P_DamageMobj(mobj_t* target, mobj_t* inflictor, mobj_t* source, int damage)
```

**Purpose:** The central damage application function. Applies a damage value to any shootable mobj, accounting for armor, god mode, invulnerability, skill level, thrust physics, and AI state changes.

**Parameters:**
- `target` - The entity receiving damage.
- `inflictor` - The physical object that caused the damage (missile or melee point); used to compute thrust direction. May be `NULL`.
- `source` - The entity that originated the attack; becomes the target's new pursuit target. May be `NULL` for environmental damage.
- `damage` - Raw damage points to apply.

**Key logic:**
1. Bails immediately if `target` is not `MF_SHOOTABLE` or is already dead.
2. Stops skull-fly momentum if the target has `MF_SKULLFLY`.
3. Baby skill: damage to players is halved (`damage >>= 1`).
4. Thrust calculation: computes angle from inflictor to target using `R_PointToAngle2`, then applies `thrust = damage*(FRACUNIT>>3)*100/mass` as momentum. Has a special "forward-fall" case for low damage that exceeds remaining health from above.
5. Chainsaw does not thrust (prevents pushing victims away).
6. For player targets:
   - End-of-game sector special 11 prevents the player from dying (caps damage at `health - 1`).
   - God mode (`CF_GODMODE`) and invulnerability powerup block damage below 1000 points.
   - Armor absorption: type 1 absorbs 1/3, type 2 absorbs 1/2. Armor is depleted and type resets to 0 when exhausted.
   - `player->damagecount` is incremented (capped at 100) to drive the red damage flash.
   - `I_Tactile()` is called for rumble feedback on the console player.
7. Subtracts damage from `target->health`. If health drops to zero or below, calls `P_KillMobj`.
8. Pain chance: if `P_Random() < target->info->painchance` and not skull-flying, transitions to pain state and sets `MF_JUSTHIT`.
9. Resets `reactiontime` to 0 (wakes the monster up).
10. If the monster has no threshold (or is an Arch-Vile), sets `target->target = source` and `threshold = BASETHRESHOLD` (100 tics of exclusive pursuit), potentially transitioning from spawn state to see state.

---

## Data Structures

No new structs are defined in this file. It operates on `player_t`, `mobj_t`, and related types defined in `d_player.h`, `p_mobj.h`, and `info.h`.

---

## Dependencies

| File | Why needed |
|------|-----------|
| `doomdef.h` | Core types (`boolean`, `fixed_t`), game constants (`MAXHEALTH`, `NUMAMMO`) |
| `dstrings.h` | Player message strings (`GOTARMOR`, `GOTSTIM`, etc.) |
| `sounds.h` | Sound effect identifiers (`sfx_itemup`, `sfx_getpow`, `sfx_wpnup`) |
| `doomstat.h` | Global game state: `gameskill`, `netgame`, `deathmatch`, `gamemode`, `players[]`, `consoleplayer` |
| `m_random.h` | `P_Random()` for randomized damage, thrust direction, and animation timing |
| `i_system.h` | `I_Error()` for fatal errors; `I_Tactile()` for haptic feedback |
| `am_map.h` | `AM_Stop()` to dismiss the automap when a player dies |
| `p_local.h` | Local play header: mobj functions, `BASETHRESHOLD`, `BONUSADD` context |
| `s_sound.h` | `S_StartSound()` to play pickup and death sounds |
| `p_inter.h` | Own header (declares `P_GivePower` as the sole exported symbol) |
