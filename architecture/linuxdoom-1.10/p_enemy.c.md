# File Overview

`p_enemy.c` is the monster AI and action-function module of the DOOM engine. It implements all enemy thinking logic, including alerting, pathfinding, attack decisions, and every monster-specific action function referenced by the state machine in `info.c`. It also implements the DOOM II Icon of Sin (brain) boss mechanics and a handful of player death/weapon sound action functions.

The file is structured into two broad categories:
1. **AI infrastructure** - Sound alerting, look/sight checks, movement, direction choosing.
2. **Action functions** (`A_*`) - Callbacks invoked by the global state machine when a monster or weapon enters a particular animation frame.

All monsters run through the state machine via `P_MobjThinker` (in `p_mobj.c`). The action functions in this file are called as side effects of state transitions.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `mobj_t*` | `soundtarget` | The mobj that triggered the current noise alert; set by `P_NoiseAlert` and propagated by `P_RecursiveSound`. |
| `fixed_t[8]` | `xspeed` | Lookup table: x-axis velocity components for each of the 8 movement directions. Values correspond to `dirtype_t` indices. |
| `fixed_t[8]` | `yspeed` | Lookup table: y-axis velocity components for each of the 8 movement directions. |
| `int` | `TRACEANGLE` | Angular correction per tic for the revenant's homing missile (`0xc000000`). |
| `mobj_t*` | `corpsehit` | Set by `PIT_VileCheck` to the corpse found eligible for resurrection by the Arch-Vile. |
| `mobj_t*` | `vileobj` | The Arch-Vile actor currently searching for corpses to raise. |
| `fixed_t` | `viletryx` | X coordinate of the Arch-Vile's prospective next position (used for blockmap corpse search). |
| `fixed_t` | `viletryy` | Y coordinate of the Arch-Vile's prospective next position. |
| `mobj_t*[32]` | `braintargets` | Array of `MT_BOSSTARGET` map things found during `A_BrainAwake`; targets for the Icon of Sin's cube spawning. |
| `int` | `numbraintargets` | Count of entries in `braintargets[]`. |
| `int` | `braintargeton` | Index of the next target in `braintargets[]` for the round-robin cube launches. |

---

## Data Structures

### `dirtype_t` (local enum)

```c
typedef enum {
    DI_EAST, DI_NORTHEAST, DI_NORTH, DI_NORTHWEST,
    DI_WEST, DI_SOUTHWEST, DI_SOUTH, DI_SOUTHEAST,
    DI_NODIR,
    NUMDIRS
} dirtype_t;
```

Represents the 8 cardinal and intercardinal directions used for monster movement. `DI_NODIR` means the monster has no valid direction and must recalculate.

### `opposite[]` and `diags[]` (file-static arrays)

- `opposite[9]`: Maps each direction to its exact opposite (used to prevent monsters from immediately reversing).
- `diags[4]`: Maps diagonal direction based on sign-of-delta combinations, used by `P_NewChaseDir`.

---

## Functions

### `void P_RecursiveSound(sector_t* sec, int soundblocks)`

**Signature:** `void P_RecursiveSound(sector_t* sec, int soundblocks)`

**Purpose:** Recursively floods adjacent sectors with a sound alert, waking monsters that can hear. Two-sided lines with the `ML_SOUNDBLOCK` flag reduce the propagation range by one "block" (only two blocks of attenuation are allowed).

**Parameters:**
- `sec` - The sector currently being processed.
- `soundblocks` - Number of sound-blocking lines already crossed (0 or 1).

**Return value:** None.

**Key logic:**
- Uses `sec->validcount` against the global `validcount` to avoid re-processing sectors in the same alert sweep.
- If already flooded with equal or better penetration (`soundtraversed <= soundblocks+1`), returns immediately.
- Sets `sec->soundtarget = soundtarget` to inform monsters in this sector of the alert target.
- For each line in the sector: if it's two-sided and open (`openrange > 0`), recurses into the adjacent sector. If the line has `ML_SOUNDBLOCK`, increments `soundblocks` (cutting off propagation after 2 blocks).

---

### `void P_NoiseAlert(mobj_t* target, mobj_t* emmiter)`

**Signature:** `void P_NoiseAlert(mobj_t* target, mobj_t* emmiter)`

**Purpose:** Initiates a noise alert radiating outward from an emitter mobj, setting all reachable sectors' `soundtarget` to `target`. Called when a player fires or a monster shouts a see-sound.

**Parameters:**
- `target` - The mobj that should become the monsters' chase target (usually the player).
- `emmiter` - The mobj generating the noise (may differ from target).

**Return value:** None.

**Key logic:** Sets the file-global `soundtarget`, increments `validcount` to start a fresh traversal, then calls `P_RecursiveSound` starting from the emitter's subsector.

---

### `boolean P_CheckMeleeRange(mobj_t* actor)`

**Signature:** `boolean P_CheckMeleeRange(mobj_t* actor)`

**Purpose:** Returns true if the actor's target is within melee attack range and in line of sight.

**Parameters:**
- `actor` - The attacking monster.

**Return value:** `true` if target is close enough for a melee hit; `false` otherwise.

**Key logic:** Computes approximate 2D distance with `P_AproxDistance`. Distance threshold is `MELEERANGE - 20*FRACUNIT + target->info->radius`. Also requires a positive `P_CheckSight()` result. The radius adjustment ensures small targets are not attacked at the same distance as large ones.

---

### `boolean P_CheckMissileRange(mobj_t* actor)`

**Signature:** `boolean P_CheckMissileRange(mobj_t* actor)`

**Purpose:** Probabilistically decides whether a monster should fire a missile at its target, based on distance, monster type, and random chance.

**Parameters:**
- `actor` - The attacking monster.

**Return value:** `true` if the monster should fire; `false` otherwise.

**Key logic:**
- Requires positive `P_CheckSight()`.
- If `MF_JUSTHIT` flag is set (just got hit), always fires (fight back) and clears the flag.
- Returns false if `actor->reactiontime > 0` (monster hasn't fully woken up).
- Computes distance and applies monster-specific corrections:
  - `MT_VILE`: Maximum effective range of 14*64 units.
  - `MT_UNDEAD` (Revenant): Will not fire if too close (prefers fist at close range).
  - `MT_CYBORG`, `MT_SPIDER`, `MT_SKULL`: Halves the distance for probability purposes (fires more readily).
- Caps effective distance at 200 (160 for Cyberdemon).
- Fires if `P_Random() >= dist` (higher dist = less likely to fire).

---

### `boolean P_Move(mobj_t* actor)`

**Signature:** `boolean P_Move(mobj_t* actor)`

**Purpose:** Attempts to move the monster one step in its current `movedir`. Handles floating monsters, special line activation by monsters, and door opening.

**Parameters:**
- `actor` - The monster to move.

**Return value:** `true` if the move succeeded (or a special was activated); `false` if blocked.

**Key logic:**
- Computes `tryx = actor->x + speed * xspeed[movedir]` and `tryy` similarly.
- Calls `P_TryMove()`. If blocked:
  - If actor has `MF_FLOAT` and `floatok` is true, adjusts height by `FLOATSPEED` toward the target level (allows flying monsters to float over obstacles).
  - Otherwise, iterates `spechit[]` (lines crossed during the failed move attempt) and calls `P_UseSpecialLine()` on each (allowing monsters to open doors).
- On success, clears `MF_INFLOAT`.
- Non-floating actors have their z snapped to `actor->floorz`.

---

### `boolean P_TryWalk(mobj_t* actor)`

**Signature:** `boolean P_TryWalk(mobj_t* actor)`

**Purpose:** Wrapper around `P_Move()`. If the move succeeds, randomizes `movecount` to prevent all monsters from changing direction simultaneously.

**Parameters:**
- `actor` - The monster to move.

**Return value:** `true` if the move succeeded; `false` if blocked.

---

### `void P_NewChaseDir(mobj_t* actor)`

**Signature:** `void P_NewChaseDir(mobj_t* actor)`

**Purpose:** Selects a new movement direction for a chasing monster using a prioritized pathfinding algorithm. This is the core of DOOM's monster navigation.

**Parameters:**
- `actor` - The monster needing a new direction.

**Return value:** None.

**Key logic (the "preferred directions" algorithm):**
1. Computes `deltax` and `deltay` to the target.
2. Derives preferred X-direction (`d[1]`) and Y-direction (`d[2]`) from the signs of the deltas.
3. **Direct diagonal attempt**: If both X and Y directions are valid and the diagonal is not a U-turn, tries `P_TryWalk()` in the diagonal direction.
4. **Axis swap randomization**: With 200/256 probability, or if `|deltay| > |deltax|`, swaps the X and Y preferred directions (adds unpredictability).
5. **Primary axis attempt**: Tries `d[1]` if not a U-turn.
6. **Secondary axis attempt**: Tries `d[2]` if not a U-turn.
7. **Old direction**: Falls back to the monster's previous direction.
8. **Random sweep**: Sweeps all 8 directions in a random order (clockwise or counterclockwise, chosen by `P_Random()&1`), trying each until one succeeds.
9. **U-turn last resort**: Finally tries the opposite direction.
10. If all fail, sets `movedir = DI_NODIR`.

---

### `boolean P_LookForPlayers(mobj_t* actor, boolean allaround)`

**Signature:** `boolean P_LookForPlayers(mobj_t* actor, boolean allaround)`

**Purpose:** Searches for a live, visible player to target. Checks up to 2 players per call in a round-robin fashion (using `actor->lastlook`) to spread the cost over multiple tics.

**Parameters:**
- `actor` - The monster searching for a target.
- `allaround` - If `false`, the monster only notices players within its 180-degree front arc (or very close ones).

**Return value:** `true` if a valid target was found and set in `actor->target`; `false` otherwise.

**Key logic:**
- Iterates active players starting at `actor->lastlook`, checking at most 2 per call.
- Skips dead players and players out of sight.
- If `allaround` is false, computes the angle to the player relative to the actor's facing angle; if the player is in the rear 180 degrees and farther than `MELEERANGE`, skips.
- Sets `actor->target` and returns `true` on success.

---

### `void A_KeenDie(mobj_t* mo)`

**Signature:** `void A_KeenDie(mobj_t* mo)`

**Purpose:** Action function for Commander Keen's death (DOOM II Map 32 secret). When the last Keen dies, opens a special door tagged 666.

**Parameters:**
- `mo` - The Keen mobj that just died.

**Return value:** None.

**Key logic:** Calls `A_Fall()`, then scans the entire thinker list. If any other live Keen mobj exists, returns immediately. If all Keens are dead, creates a dummy `line_t` with `tag = 666` and calls `EV_DoDoor(&junk, open)`.

---

### `void A_Look(mobj_t* actor)`

**Signature:** `void A_Look(mobj_t* actor)`

**Purpose:** State action for the monster's idle/spawn state. Looks for players to target; transitions to chase state on success.

**Parameters:**
- `actor` - The monster in its idle state.

**Return value:** None.

**Key logic:**
- Resets `actor->threshold = 0`.
- Checks `sector->soundtarget`: if a shootable target exists in the sector's sound field, immediately acquires it. Ambush monsters additionally require sight before activating.
- Falls back to `P_LookForPlayers(actor, false)` (frontal arc only).
- On acquiring a target, plays the see-sound (with random selection for Possessed/Sergeant), transitions to `actor->info->seestate` via `P_SetMobjState()`.

---

### `void A_Chase(mobj_t* actor)`

**Signature:** `void A_Chase(mobj_t* actor)`

**Purpose:** Core chase-state action function. Called each frame in the chase state to decrement reaction time, turn toward movement direction, check for and execute melee or missile attacks, navigate toward the target, and play idle sounds.

**Parameters:**
- `actor` - The chasing monster.

**Return value:** None.

**Key logic:**
1. Decrements `reactiontime` if nonzero.
2. Decrements `threshold` if nonzero (while threshold > 0, the monster locks onto its current target even in netgame).
3. Gradually aligns `actor->angle` toward `movedir` (by ANG90/2 per tic).
4. If no valid shootable target, calls `P_LookForPlayers(actor, true)` for a new one; on failure, reverts to spawn state.
5. Handles `MF_JUSTATTACKED`: skips one movement tick after attacking.
6. **Melee attack**: If in melee range and the monster has a melee state, transitions to it.
7. **Missile attack**: If conditions allow (not nightmare + already moved, or missile range check passes), transitions to missile state and sets `MF_JUSTATTACKED`.
8. **Movement**: Decrements `movecount`. When exhausted, or when `P_Move()` fails, calls `P_NewChaseDir()`.
9. **Active sound**: 3/256 chance of playing the monster's active sound.

---

### `void A_FaceTarget(mobj_t* actor)`

**Signature:** `void A_FaceTarget(mobj_t* actor)`

**Purpose:** Rotates the actor to face its current target. Clears `MF_AMBUSH`. Adds angular noise when targeting a shadow (spectre/partial-invisibility) mobj.

**Parameters:**
- `actor` - The monster.

**Return value:** None.

---

### `void A_PosAttack(mobj_t* actor)`

**Purpose:** Former Human (Possessed) single-shot hitscan attack. Faces target, aims with `P_AimLineAttack`, fires one hitscan bullet with random angular spread (`<<20`) and random damage (3-15 points, multiples of 3).

---

### `void A_SPosAttack(mobj_t* actor)`

**Purpose:** Former Sergeant (Shotgun Guy) attack. Fires 3 hitscan pellets simultaneously, each with independent random spread, simulating a shotgun blast (3-15 damage each, sharing the same vertical aim slope).

---

### `void A_CPosAttack(mobj_t* actor)`

**Purpose:** Heavy Weapon Dude (Chaingunner) single-shot attack. One hitscan bullet per call with spread; the Chaingunner fires multiple times per attack sequence by having multiple frames call this.

---

### `void A_CPosRefire(mobj_t* actor)`

**Purpose:** Chaingunner refire check. Called between bursts; with 40/256 probability, continues firing without checking. Otherwise stops if the target is dead or out of sight.

---

### `void A_SpidRefire(mobj_t* actor)`

**Purpose:** Spider Mastermind refire check. Like `A_CPosRefire` but uses a 10/256 threshold (fires much more aggressively).

---

### `void A_BspiAttack(mobj_t* actor)`

**Purpose:** Arachnotron attack. Faces target and fires an `MT_ARACHPLAZ` (plasma bolt) missile via `P_SpawnMissile`.

---

### `void A_TroopAttack(mobj_t* actor)`

**Purpose:** Imp attack. If in melee range, performs a claw attack (random 3-24 damage, `sfx_claw`). Otherwise launches an `MT_TROOPSHOT` (fireball) missile.

---

### `void A_SargAttack(mobj_t* actor)`

**Purpose:** Demon/Spectre melee attack. Only attacks in melee range: 4-40 random damage.

---

### `void A_HeadAttack(mobj_t* actor)`

**Purpose:** Cacodemon attack. Melee if in range (10-60 damage), otherwise fires `MT_HEADSHOT` (blue ball) missile.

---

### `void A_CyberAttack(mobj_t* actor)`

**Purpose:** Cyberdemon attack. Always fires an `MT_ROCKET` missile at the target.

---

### `void A_BruisAttack(mobj_t* actor)`

**Purpose:** Baron of Hell / Hell Knight attack. Melee if in range (10-80 damage, `sfx_claw`), otherwise fires `MT_BRUISERSHOT` (green fireball).

---

### `void A_SkelMissile(mobj_t* actor)`

**Signature:** `void A_SkelMissile(mobj_t* actor)`

**Purpose:** Revenant missile attack. Temporarily raises the actor's Z by 16 units before spawning the missile so it fires from shoulder height, then restores Z. Sets `mo->tracer` to the target so `A_Tracer` can home on it.

**Parameters:**
- `actor` - The Revenant.

---

### `void A_Tracer(mobj_t* actor)`

**Signature:** `void A_Tracer(mobj_t* actor)`

**Purpose:** Per-tic homing correction for the Revenant's tracer missile. Active every 4 tics (`gametic & 3`). Spawns a smoke trail particle and corrects both the horizontal angle and vertical slope toward the target.

**Parameters:**
- `actor` - The tracer missile mobj.

**Return value:** None.

**Key logic:**
- Spawns `MT_SMOKE` behind the missile.
- Computes angle from missile to `actor->tracer` (the target). If the angle differs from current heading, rotates by `TRACEANGLE` (0xc000000) per call, clamped to exact angle when overshoot would occur.
- Updates `momx`/`momy` from the new angle and `actor->info->speed`.
- Computes vertical distance to the target center (target->z + 40 units) and adjusts `momz` by FRACUNIT/8 per tic toward the correct slope.

---

### `void A_SkelWhoosh(mobj_t* actor)`

**Purpose:** Revenant melee windup sound (`sfx_skeswg`) and face-target.

---

### `void A_SkelFist(mobj_t* actor)`

**Purpose:** Revenant melee hit. If in melee range, deals 6-60 damage with `sfx_skepch`.

---

### `boolean PIT_VileCheck(mobj_t* thing)`

**Signature:** `boolean PIT_VileCheck(mobj_t* thing)`

**Purpose:** Blockmap iterator callback used by `A_VileChase`. Checks whether a map object is a resurrect-able corpse within range of the Arch-Vile.

**Parameters:**
- `thing` - The mobj in the current blockmap cell.

**Return value:** `true` to continue iteration; `false` (stop) when a valid corpse is found.

**Key logic:**
- Must have `MF_CORPSE`, `tics == -1` (fully dead, not in death animation), and `info->raisestate != S_NULL`.
- Checks approximate bounding-box distance (`viletryx`/`viletryy` against the corpse position, using combined radii).
- Temporarily inflates the corpse's height by 4x and calls `P_CheckPosition()` to verify there is room to resurrect it.
- If valid: sets `corpsehit = thing`, stops `momy`/`momx`, returns `false` (found one).

---

### `void A_VileChase(mobj_t* actor)`

**Signature:** `void A_VileChase(mobj_t* actor)`

**Purpose:** Arch-Vile chase action. Searches a 2-radius blockmap area ahead of the Vile for resurrect-able corpses. If one is found, triggers the resurrection animation and restores the corpse's health/flags/state.

**Parameters:**
- `actor` - The Arch-Vile.

**Return value:** None.

**Key logic:**
- Computes the Arch-Vile's prospective next position (`viletryx`, `viletryy`) using `movedir`/`xspeed`/`yspeed`.
- Scans blockmap cells in a 2*MAXRADIUS box using `P_BlockThingsIterator(bx, by, PIT_VileCheck)`.
- If a corpse is found (`PIT_VileCheck` returned false, setting `corpsehit`):
  - Temporarily sets `actor->target` to the corpse and calls `A_FaceTarget()`.
  - Restores `actor->target` to the original target.
  - Puts actor in `S_VILE_HEAL1` state, plays `sfx_slop`.
  - Calls `P_SetMobjState(corpsehit, info->raisestate)`, inflates height by 4x, restores flags and health.
- If no corpse found, falls through to `A_Chase`.

---

### `void A_VileStart(mobj_t* actor)`

**Purpose:** Plays the Arch-Vile attack windup sound (`sfx_vilatk`).

---

### `void A_StartFire(mobj_t* actor)`

**Purpose:** Plays `sfx_flamst` and calls `A_Fire` to position the hell-fire object.

---

### `void A_FireCrackle(mobj_t* actor)`

**Purpose:** Plays `sfx_flame` and calls `A_Fire`.

---

### `void A_Fire(mobj_t* actor)`

**Signature:** `void A_Fire(mobj_t* actor)`

**Purpose:** Repositions the Arch-Vile's hell-fire object so it stays in front of the targeted player.

**Parameters:**
- `actor` - The fire mobj (its `tracer` is the Vile, its `target` is the Vile's target player).

**Return value:** None.

**Key logic:** Checks that the Arch-Vile (`actor->target`) still has sight to `dest` (`actor->tracer`, the player). If so, positions the fire 24 units in front of the destination object using its angle.

---

### `void A_VileTarget(mobj_t* actor)`

**Signature:** `void A_VileTarget(mobj_t* actor)`

**Purpose:** Spawns the `MT_FIRE` hell-fire object at the player's position and links it to the Arch-Vile.

**Parameters:**
- `actor` - The Arch-Vile.

**Note:** Contains a known bug: the Y coordinate for spawning uses `actor->target->x` instead of `actor->target->y`.

---

### `void A_VileAttack(mobj_t* actor)`

**Signature:** `void A_VileAttack(mobj_t* actor)`

**Purpose:** Executes the Arch-Vile's final attack: deals 20 direct damage, launches the target upward, and triggers a 70-damage radius blast from the fire object's position.

**Parameters:**
- `actor` - The Arch-Vile.

**Key logic:**
- Requires line of sight to target.
- Plays `sfx_barexp`.
- Applies `momz = 1000*FRACUNIT / mass` to launch the player upward.
- Repositions the fire in front of the target and calls `P_RadiusAttack(fire, actor, 70)`.

---

### `void A_FatRaise(mobj_t* actor)`

**Purpose:** Mancubus pre-attack animation: face target and play `sfx_manatk`.

---

### `void A_FatAttack1(mobj_t* actor)`

**Purpose:** Mancubus attack 1. Fires two `MT_FATSHOT` (fireblob) missiles: one straight, one spread by `+FATSPREAD` (ANG90/8) in angle.

---

### `void A_FatAttack2(mobj_t* actor)`

**Purpose:** Mancubus attack 2. Fires two `MT_FATSHOT` missiles with spread in the negative direction: one straight, one at `-FATSPREAD*2`.

---

### `void A_FatAttack3(mobj_t* actor)`

**Purpose:** Mancubus attack 3. Fires two `MT_FATSHOT` missiles bracketing center: one at `-FATSPREAD/2`, one at `+FATSPREAD/2`.

---

### `void A_SkullAttack(mobj_t* actor)`

**Signature:** `void A_SkullAttack(mobj_t* actor)`

**Purpose:** Lost Soul charge attack. Sets `MF_SKULLFLY`, computes velocity toward the target at `SKULLSPEED` (20 map units/tic), and calculates `momz` to intercept the target vertically.

**Parameters:**
- `actor` - The Lost Soul.

---

### `void A_PainShootSkull(mobj_t* actor, angle_t angle)`

**Signature:** `void A_PainShootSkull(mobj_t* actor, angle_t angle)`

**Purpose:** Spawns a new Lost Soul at the given angle relative to the Pain Elemental and launches it at the current target. Limited to 20 active skulls on the level.

**Parameters:**
- `actor` - The Pain Elemental.
- `angle` - Direction to spit the skull.

**Return value:** None.

**Key logic:**
- Counts existing `MT_SKULL` thinkers; returns without spawning if >20.
- Computes spawn position `prestep` units ahead of the actor along `angle`.
- Spawns the skull at the computed position; if it cannot fit (`P_TryMove` fails), instantly kills it with 10000 damage.
- Sets the skull's target and calls `A_SkullAttack()` on it.

---

### `void A_PainAttack(mobj_t* actor)`

**Purpose:** Pain Elemental frontal attack: face target and spit one skull forward.

---

### `void A_PainDie(mobj_t* actor)`

**Purpose:** Pain Elemental death: calls `A_Fall`, then spits three skulls at +90, +180, +270 degree offsets.

---

### `void A_Scream(mobj_t* actor)`

**Purpose:** Monster death scream. Randomizes the sound for Possessed (3 variants) and Demon (2 variants). Spider Mastermind and Cyberdemon play at full volume (NULL origin).

---

### `void A_XScream(mobj_t* actor)`

**Purpose:** Gibbing death sound (`sfx_slop`).

---

### `void A_Pain(mobj_t* actor)`

**Purpose:** Plays the monster's pain sound if it has one.

---

### `void A_Fall(mobj_t* actor)`

**Purpose:** Clears `MF_SOLID` when a monster falls, making the corpse walkable.

---

### `void A_Explode(mobj_t* thingy)`

**Purpose:** Action for barrel/missile explosions: calls `P_RadiusAttack(thingy, thingy->target, 128)`.

---

### `void A_BossDeath(mobj_t* mo)`

**Signature:** `void A_BossDeath(mobj_t* mo)`

**Purpose:** Checks whether a boss's death should trigger a map event (lower floor, open door, exit level). Implements all per-episode boss-kill conditions for Doom and Doom II.

**Parameters:**
- `mo` - The boss mobj that just entered its death state.

**Return value:** None.

**Key logic:**
- In Doom II commercial mode: only triggers on Map 07 when all Mancubi (tag 666 floor lower) or all Arachnotrons (tag 667 floor raise) are dead.
- In registered/retail mode:
  - Episode 1 Map 8: All Barons of Hell dead → lower floor (tag 666).
  - Episode 2 Map 8: Cyberdemon dead → exit level.
  - Episode 3 Map 8: Spider Mastermind dead → exit level.
  - Episode 4 Map 6: Cyberdemon dead → open door (tag 666). Map 8: Spider Mastermind dead → lower floor.
- Ensures at least one player is alive before triggering victory.
- Scans all thinkers for remaining boss mobjtype; if any survive, does not trigger.

---

### `void A_Hoof(mobj_t* mo)` / `void A_Metal(mobj_t* mo)` / `void A_BabyMetal(mobj_t* mo)`

**Purpose:** Walking sound action functions. Play hoof, metal, and baby-step sounds respectively, then call `A_Chase`. Used for Cyberdemon, Spider Mastermind, and Arachnotron footstep sounds.

---

### `void A_OpenShotgun2(player_t* player, pspdef_t* psp)`

**Purpose:** Super shotgun break-open sound (`sfx_dbopn`).

---

### `void A_LoadShotgun2(player_t* player, pspdef_t* psp)`

**Purpose:** Super shotgun reload/shell-load sound (`sfx_dbload`).

---

### `void A_CloseShotgun2(player_t* player, pspdef_t* psp)`

**Purpose:** Super shotgun close sound (`sfx_dbcls`), then calls `A_ReFire` to continue firing if button is held.

---

### `void A_BrainAwake(mobj_t* mo)`

**Signature:** `void A_BrainAwake(mobj_t* mo)`

**Purpose:** Scans the thinker list to find all `MT_BOSSTARGET` things and stores them in `braintargets[]`. Plays the `sfx_bossit` sound. Called when the Icon of Sin first activates.

---

### `void A_BrainPain(mobj_t* mo)`

**Purpose:** Plays `sfx_bospn` when the Icon of Sin takes damage.

---

### `void A_BrainScream(mobj_t* mo)`

**Purpose:** Spawns a row of rocket objects that explode along the ceiling, creating the dramatic death explosion effect for the Icon of Sin.

---

### `void A_BrainExplode(mobj_t* mo)`

**Purpose:** Spawns one additional explosion rocket per call during the Icon of Sin death sequence.

---

### `void A_BrainDie(mobj_t* mo)`

**Purpose:** Calls `G_ExitLevel()` to end DOOM II when the Icon of Sin is destroyed.

---

### `void A_BrainSpit(mobj_t* mo)`

**Signature:** `void A_BrainSpit(mobj_t* mo)`

**Purpose:** Icon of Sin's cube-launching attack. On easy difficulty, only fires every other call. Picks the next target from `braintargets[]` in round-robin order and fires an `MT_SPAWNSHOT` cube toward it.

**Parameters:**
- `mo` - The Icon of Sin brain mobj.

**Return value:** None.

**Key logic:**
- Uses a static `easy` toggle to skip every other shot on easy skill.
- Computes `reactiontime` for the cube as `(distance / momy) / state->tics` so the cube arrives in time.

---

### `void A_SpawnSound(mobj_t* mo)`

**Purpose:** Plays `sfx_boscub` (cube-in-flight sound) and calls `A_SpawnFly`.

---

### `void A_SpawnFly(mobj_t* mo)`

**Signature:** `void A_SpawnFly(mobj_t* mo)`

**Purpose:** Per-tic update for the flying spawn cube. Counts down `reactiontime`; when it reaches 0, spawns a random monster at the target location, telefragging anything there, and removes the cube.

**Parameters:**
- `mo` - The spawn cube mobj.

**Return value:** None.

**Key logic:**
- Spawns teleport fog at the target spot, plays `sfx_telept`.
- Selects monster type using `P_Random()` probability distribution:
  - r < 50: Imp, r < 90: Demon, r < 120: Spectre, r < 130: Pain Elemental,
  - r < 160: Cacodemon, r < 162: Arch-Vile, r < 172: Revenant, r < 192: Arachnotron,
  - r < 222: Mancubus, r < 246: Hell Knight, else: Baron of Hell.
- Calls `P_TeleportMove()` to telefrag anything at the destination.
- Removes the cube with `P_RemoveMobj`.

---

### `void A_PlayerScream(mobj_t* mo)`

**Purpose:** Player death sound. Uses `sfx_pldeth` normally; switches to `sfx_pdiehi` (Doom II only) when health drops below -50 (ultra-gib death).

---

## Dependencies

| File | Purpose |
|------|---------|
| `stdlib.h` | Standard library (abs) |
| `m_random.h` | `P_Random()` for all random decisions |
| `i_system.h` | `I_Error()` for fatal errors |
| `doomdef.h` | Core types, `MELEERANGE`, `MISSILERANGE`, `FRACUNIT` |
| `p_local.h` | All play subsystem APIs: `P_TryMove`, `P_CheckPosition`, `P_CheckSight`, `P_AimLineAttack`, `P_LineAttack`, `P_SpawnMissile`, `P_DamageMobj`, `P_RadiusAttack`, `P_BlockThingsIterator`, `P_SetMobjState`, `P_SpawnMobj`, `P_RemoveMobj`, `P_SpawnPuff`, `P_NoiseAlert`, `P_UseSpecialLine`, `P_TeleportMove`, blockmap globals |
| `s_sound.h` | `S_StartSound()` |
| `g_game.h` | `G_ExitLevel()` |
| `doomstat.h` | `gameskill`, `netgame`, `fastparm`, `gametic`, `leveltime`, `gameepisode`, `gamemap`, `gamemode`, `respawnmonsters`, `players[]`, `playeringame[]`, `thinkercap` |
| `r_state.h` | `validcount`, `finecosine[]`, `finesine[]`, `bmaporgx/y`, `MAPBLOCKSHIFT`, `MAXRADIUS` |
| `sounds.h` | Sound effect enumerations |
