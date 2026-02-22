# File Overview

`d_player.h` defines the complete player state structure (`player_t`) and its supporting types, as well as the intermission data structures used to pass statistics between the level-end sequence and the world-map/intermission screen. It is one of the most widely included headers in the engine, since every subsystem that needs to read or modify player state must include it.

The `player_t` structure is a "fat" record capturing everything about a player: physics state (linked to the map object `mobj_t`), inventory (weapons, ammo, keys, backpack, power-ups), HUD/visual state (view height, color map, gun sprites), input buffering (the current `ticcmd_t`), multiplayer bookkeeping (frags, messages), and intermission statistics. It is distinct from `mobj_t` (which handles physical simulation); `player_t` is the "human" layer on top.

## Global Variables

This header defines no global variables. The `players[MAXPLAYERS]` array is declared in `doomstat.h` and defined in `g_game.c`.

## Functions

No functions are declared in this header. It is purely a type-definition file.

## Data Structures

### `playerstate_t` (enum)

Describes the current life/death state of a player.

| Value | Meaning |
|-------|---------|
| `PST_LIVE` | Player is alive and active. |
| `PST_DEAD` | Player is dead; the view follows the killer. |
| `PST_REBORN` | Player is ready to respawn (triggers `G_DoReborn`). |

---

### `cheat_t` (enum)

Bit flags stored in `player_t::cheats` for active cheat states.

| Flag | Value | Effect |
|------|-------|--------|
| `CF_NOCLIP` | `1` | No clipping — player walks through walls. |
| `CF_GODMODE` | `2` | Invulnerable — no health loss. |
| `CF_NOMOMENTUM` | `4` | No momentum (debug aid — player does not drift). |

---

### `player_t` (struct)

The complete per-player state record. Defined as `player_s` with typedef `player_t`.

```c
typedef struct player_s {
    mobj_t*       mo;                        // Physical map object.
    playerstate_t playerstate;               // Live, dead, or ready to reborn.
    ticcmd_t      cmd;                       // Current tic command (input for this tick).

    fixed_t       viewz;                     // Camera height in world units.
    fixed_t       viewheight;                // Base height above floor for viewz.
    fixed_t       deltaviewheight;           // Bob/squat speed adjustment.
    fixed_t       bob;                       // Scaled total momentum for view bobbing.

    int           health;                    // Health between levels (mo->health used during play).
    int           armorpoints;               // Current armor value.
    int           armortype;                 // 0 = none, 1 = green, 2 = blue armor.

    int           powers[NUMPOWERS];         // Power-up tic counters (0 = inactive).
    boolean       cards[NUMCARDS];           // Which key cards/skulls are held.
    boolean       backpack;                  // True if player has picked up the backpack.

    int           frags[MAXPLAYERS];         // Frag counts against each other player.
    weapontype_t  readyweapon;               // Currently wielded weapon.
    weapontype_t  pendingweapon;             // Weapon being switched to (wp_nochange = none).

    boolean       weaponowned[NUMWEAPONS];   // Which weapons the player possesses.
    int           ammo[NUMAMMO];             // Current ammo counts per type.
    int           maxammo[NUMAMMO];          // Maximum ammo capacity per type.

    int           attackdown;                // True if fire button was held last tic.
    int           usedown;                   // True if use button was held last tic.

    int           cheats;                    // Active cheat flags (see cheat_t).

    int           refire;                    // Refired-shot counter; higher = less accurate.

    int           killcount;                 // Enemies killed (for intermission/stats).
    int           itemcount;                 // Items collected.
    int           secretcount;              // Secrets found.

    char*         message;                   // Current HUD hint/message string pointer.

    int           damagecount;               // Tics of red screen flash remaining.
    int           bonuscount;               // Tics of yellow bonus flash remaining.

    mobj_t*       attacker;                  // The mobj that last damaged this player (NULL = environment).

    int           extralight;               // Additional light level boost (gun flash).
    int           fixedcolormap;            // Fixed palette/colormap index (invulnerability, IR goggles).
    int           colormap;                 // Player skin color shift (0-3, multiplayer).

    pspdef_t      psprites[NUMPSPRITES];    // Overlay weapon/gun sprites.

    boolean       didsecret;               // True if the player has completed a secret level.
} player_t;
```

Field notes:
- `viewz` is computed each frame from `viewheight` and the floor height of the player's current sector. It varies due to `bob`.
- `bob` is derived from the player's velocity and causes the classic view-bobbing while walking. Maximum is capped.
- `powers[]` stores remaining tic counts for time-limited power-ups (invulnerability, strength, invisibility, iron feet, allmap, infrared). A value of 0 means inactive; the constant `NUMPOWERS` is defined in `doomdef.h`.
- `psprites` holds the animated weapon overlays. `NUMPSPRITES` is typically 2 (weapon and muzzle flash), defined in `p_pspr.h`.
- `fixedcolormap` is set to `INVERSECOLORMAP` during invulnerability (causes the distinctive white-out effect) and to an infrared map during the radiation suit power-up.

---

### `wbplayerstruct_t` (struct)

Per-player statistics passed to the intermission (`WI_Start`).

```c
typedef struct {
    boolean in;        // True if this player is participating in the game.
    int     skills;    // Kill count at level end.
    int     sitems;    // Item count at level end.
    int     ssecret;   // Secret count at level end.
    int     stime;     // Level time in tics.
    int     frags[4];  // Frag counts (multiplayer).
    int     score;     // Score on entry; modified on return.
} wbplayerstruct_t;
```

---

### `wbstartstruct_t` (struct)

Top-level structure passed to `WI_Start()` to control the world map and intermission display.

```c
typedef struct {
    int  epsd;          // Episode number (0-based).
    boolean didsecret;  // True if the console player completed a secret level.
    int  last;          // Previous map number (0-based).
    int  next;          // Next map number (0-based).
    int  maxkills;      // Total killable enemies on the level.
    int  maxitems;      // Total collectable items.
    int  maxsecret;     // Total secrets.
    int  maxfrags;      // Maximum frags (multiplayer).
    int  partime;       // Par time in tics.
    int  pnum;          // Console player's index.
    wbplayerstruct_t plyr[MAXPLAYERS]; // Per-player stats.
} wbstartstruct_t;
```

## Dependencies

| File | Reason |
|------|--------|
| `d_items.h` | `weapontype_t`, `ammotype_t`, `NUMWEAPONS`, `NUMAMMO` — weapon and ammo type enums used in `player_t`. |
| `p_pspr.h` | `pspdef_t`, `NUMPSPRITES` — weapon overlay sprite definition used in `player_t`. |
| `p_mobj.h` | `mobj_t` — the physical map object type pointed to by `player_t::mo`. |
| `d_ticcmd.h` | `ticcmd_t` — per-tick input command embedded in `player_t::cmd`. |
| `doomdef.h` | (indirectly through the above) `MAXPLAYERS`, `NUMCARDS`, `NUMPOWERS`, `fixed_t`, `boolean`. |
