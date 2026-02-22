# File Overview

`p_user.c` handles all per-tic player-specific logic that runs inside `P_PlayerThink`. This encompasses:

- **Thrust/movement**: Translating player input commands into momentum on the map object.
- **View height and bobbing**: Computing the camera's vertical position including the characteristic walking bob, landing/injury view dip, and death fall animation.
- **Player movement application**: Reading the `ticcmd_t` command for the current tic and applying angle turns, forward/side thrusts, and transitioning to running animation states.
- **Death processing**: Orienting the dead player's view toward their killer and waiting for a USE press to respawn.
- **The master `P_PlayerThink` function**: The per-tic entry point that processes all player state: noclip cheat, reaction time, movement, height calculation, special sector interaction, weapon changing, USE button, weapon sprite animation, and power-up countdowns/colormaps.

## Global Variables

| Type | Name | Description |
|---|---|---|
| `boolean` | `onground` | Set to `true` when the player's map object `z` equals `floorz` (player is standing on the floor). Used by `P_MovePlayer` to suppress air-control and by `P_CalcHeight`/`P_DeathThink` for bob calculations. Module-scoped but referenced by name in the same translation unit. |

### Compile-time constants (not variables, but architecturally significant)

| Constant | Value | Meaning |
|---|---|---|
| `INVERSECOLORMAP` | `32` | Index into the colormap array for the invulnerability inverse-video effect. |
| `MAXBOB` | `0x100000` | Maximum bob amplitude in fixed-point units (16 pixels of vertical displacement). |
| `ANG5` | `ANG90/18` | Five degrees in DOOM's binary angle units; used as the turn rate when rotating toward the killer during death. |

## Functions

### `P_Thrust`

```c
void P_Thrust(player_t* player, angle_t angle, fixed_t move)
```

**Purpose:** Applies a horizontal impulse to the player's map object along the given world angle. Used for both forward/backward and side-strafing movement.

**Parameters:**
- `player` (`player_t*`): The player whose `mo->momx` and `mo->momy` are modified.
- `angle` (`angle_t`): Direction of thrust in DOOM binary angle units (0 = east).
- `move` (`fixed_t`): Magnitude of the thrust in fixed-point units.

**Return Value:** `void`

**Key Logic:** Shifts `angle` right by `ANGLETOFINESHIFT` to convert from the 32-bit angle space to an index into the `finecosine`/`finesine` lookup tables (8192 entries covering one full rotation). Adds `FixedMul(move, finecosine[angle])` to `momx` and `FixedMul(move, finesine[angle])` to `momy`. Note that momentum is **accumulated** — multiple thrusts per tic are additive before friction is applied.

---

### `P_CalcHeight`

```c
void P_CalcHeight(player_t* player)
```

**Purpose:** Computes the player's `viewz` (camera height) and `viewheight` (eye offset above `mo->z`) for the current tic, including the walking bob effect and view-dip recovery after taking damage or landing.

**Parameters:**
- `player` (`player_t*`): The player structure to update.

**Return Value:** `void`

**Key Logic:**

1. **Bob calculation:** Computes `player->bob` from the square of horizontal momentum: `bob = (momx*momx + momy*momy) >> 2`, clamped to `MAXBOB`. This gives a bob amplitude proportional to speed.

2. **Noclip / air:** If the player has the `CF_NOMOMENTUM` cheat or is not on the ground (`!onground`), sets `viewz = mo->z + viewheight` (no bob) and returns early.

3. **Bob sine wave:** Computes a time-varying bob offset using `finesine[(FINEANGLES/20*leveltime) & FINEMASK]` scaled by half the bob amplitude. The period is 20 tics at full speed.

4. **View height damping:** If alive (`PST_LIVE`), adjusts `viewheight` by `deltaviewheight` (springs back toward `VIEWHEIGHT` of 41 units). Enforces minimum of `VIEWHEIGHT/2` with a corrective `deltaviewheight = 1` to prevent negative heights. `deltaviewheight` accelerates by `FRACUNIT/4` per tic.

5. **Final viewz:** `viewz = mo->z + viewheight + bob`, clamped to `mo->ceilingz - 4*FRACUNIT` to avoid clipping through low ceilings.

---

### `P_MovePlayer`

```c
void P_MovePlayer(player_t* player)
```

**Purpose:** Reads the player's current `ticcmd_t` and applies angle turns and directional thrusts to the player's map object.

**Parameters:**
- `player` (`player_t*`): The player to move.

**Return Value:** `void`

**Key Logic:**

1. **Angle turn:** `mo->angle += cmd->angleturn << 16`. The `ticcmd_t` angle turn is a signed 16-bit value (from mouse/keyboard); shifting left 16 converts it to the 32-bit angle space.

2. **Ground check:** `onground = (mo->z <= mo->floorz)`. Air-control is not permitted.

3. **Forward thrust:** If `cmd->forwardmove` and on ground, calls `P_Thrust(player, mo->angle, cmd->forwardmove * 2048)`. The scale factor 2048 converts the byte-sized command value to a meaningful fixed-point force.

4. **Side thrust:** If `cmd->sidemove` and on ground, calls `P_Thrust(player, mo->angle - ANG90, cmd->sidemove * 2048)`. Subtracting `ANG90` from the facing angle gives the rightward direction.

5. **Run animation:** If any movement occurred and the player state is `S_PLAY` (standing idle), transitions to `S_PLAY_RUN1` via `P_SetMobjState`.

---

### `P_DeathThink`

```c
void P_DeathThink(player_t* player)
```

**Purpose:** Per-tic processing for a dead player. Animates the view falling to floor level, slowly rotates the camera toward the killer, decrements the damage flash, and waits for the USE button to initiate respawn.

**Parameters:**
- `player` (`player_t*`): The dead player.

**Return Value:** `void`

**Key Logic:**

1. **Weapon sprites:** Calls `P_MovePsprites(player)` so weapon animations can still run (e.g., the death animation for the currently held weapon).

2. **View fall:** Decrements `player->viewheight` by `FRACUNIT` per tic until it reaches the floor minimum of `6*FRACUNIT` (~6 pixels above the ground).

3. **Killer tracking:** If `player->attacker` exists and is not the player themselves, computes the angle from the player's position to the attacker using `R_PointToAngle2`. Rotates the player's angle toward the attacker by `ANG5` per tic (or instantly snaps if within `ANG5`). When facing the killer, decrements `damagecount`.

4. **Respawn:** If `cmd->buttons & BT_USE`, sets `player->playerstate = PST_REBORN` to trigger level restart.

---

### `P_PlayerThink`

```c
void P_PlayerThink(player_t* player)
```

**Purpose:** The per-tic entry point for all player logic. Called by `P_Ticker` for each active player every game tic.

**Parameters:**
- `player` (`player_t*`): The player to process.

**Return Value:** `void`

**Key Logic (in order of execution):**

1. **Noclip cheat:** Syncs `MF_NOCLIP` flag on `mo` to match `CF_NOCLIP` cheat flag.

2. **Chainsaw auto-run:** If `MF_JUSTATTACKED` is set (set by the chainsaw attack code), forces `forwardmove = 0xc800/512` and clears the flag.

3. **Death dispatch:** If `playerstate == PST_DEAD`, calls `P_DeathThink` and returns.

4. **Reaction time:** If `mo->reactiontime > 0`, decrements it and skips movement (freeze after teleport). Otherwise calls `P_MovePlayer`.

5. **Height calculation:** Always calls `P_CalcHeight` to update camera position.

6. **Special sector:** If standing in a sector with a `special` type (damaging floor, exit, etc.), calls `P_PlayerInSpecialSector`.

7. **Weapon change:** Processes `BT_CHANGE` button. Handles the fist-to-chainsaw and shotgun-to-super-shotgun promotions. Also blocks plasma/BFG selection in shareware mode. Sets `player->pendingweapon` (actual switch happens in weapon sprite state machine).

8. **USE button:** On first press (`!player->usedown`), calls `P_UseLines` to interact with linedefs. Tracks button hold state in `player->usedown` to prevent repeat triggering.

9. **Weapon sprite tick:** Calls `P_MovePsprites` to advance weapon animation states.

10. **Power-up countdowns:**
    - `pw_strength` counts **up** (berserk fade-out: colormap goes from red to normal as the counter grows beyond visible range).
    - `pw_invulnerability`, `pw_invisibility`, `pw_infrared`, `pw_ironfeet` count **down**.
    - When `pw_invisibility` hits zero, clears `MF_SHADOW` on `mo`.
    - `damagecount` and `bonuscount` also count down (screen-tint effects).

11. **Colormap selection:**
    - Invulnerability: uses `INVERSECOLORMAP` (color index 32, the inverse palette) while either more than 128 tics remain or the counter's bit 3 is set (flashing effect as it runs out).
    - Infrared (light amplification): uses colormap index 1 (near-maximum brightness) under the same flashing condition.
    - Otherwise: `fixedcolormap = 0` (normal sector-based lighting).

## Data Structures

No new data structures are defined in this file. It operates on the pre-existing player and game types defined in `d_player.h`, `p_local.h`, and `doomdef.h`.

## Dependencies

| File | Reason |
|---|---|
| `doomdef.h` | Core type definitions, `FRACUNIT`, `VIEWHEIGHT`, angle constants |
| `d_event.h` | `ticcmd_t` definition (button flags `BT_USE`, `BT_CHANGE`, `BT_SPECIAL`, `BT_WEAPONMASK`, `BT_WEAPONSHIFT`) |
| `p_local.h` | `P_Thrust`, `P_MovePlayer`, `P_CalcHeight`, `P_SetMobjState`, `P_MovePsprites`, `P_UseLines`, `P_PlayerInSpecialSector`, `ANG90`, and all play-module internals |
| `doomstat.h` | `leveltime`, `gamemode` (shareware vs. commercial check), `players[]` array |
