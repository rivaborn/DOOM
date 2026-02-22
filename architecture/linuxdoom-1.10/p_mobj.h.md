# File Overview

**Source file:** `linuxdoom-1.10/p_mobj.h`

`p_mobj.h` is one of the most central headers in the DOOM engine. It defines the `mobj_t` structure — the universal representation of every interactive entity in the game world — together with the complete `mobjflag_t` enumeration that governs object behaviour. It also provides extensive inline documentation explaining the architecture of the object system, the sector/blockmap linking scheme, and the purpose of each flag.

This header is included by nearly every subsystem that needs to read or write mobj data: the renderer (to find sprite/frame info), the physics code, the AI, the sound system, the save/load system, and the game logic.

---

## Data Structures

### `mobjflag_t` (enum)

The behaviour flags stored in `mobj_t::flags`. Multiple flags may be OR'd together.

| Flag | Value | Purpose |
|------|-------|---------|
| `MF_SPECIAL` | `0x00000001` | Triggers `P_SpecialThing` when a player touches the object (pickup items). |
| `MF_SOLID` | `0x00000002` | Blocks movement; other objects cannot pass through it. |
| `MF_SHOOTABLE` | `0x00000004` | Can be damaged by attacks. |
| `MF_NOSECTOR` | `0x00000008` | Not linked into the sector thing list. Invisible to the renderer but still touchable. |
| `MF_NOBLOCKMAP` | `0x00000010` | Not linked into the blockmap. Inert to collision but still displayable. |
| `MF_AMBUSH` | `0x00000020` | Deaf; will not be activated by sound. Must see the player to wake. |
| `MF_JUSTHIT` | `0x00000040` | Recently hit; will try to attack back immediately. |
| `MF_JUSTATTACKED` | `0x00000080` | Just attacked; will take at least one step before attacking again. |
| `MF_SPAWNCEILING` | `0x00000100` | Spawns hanging from the ceiling rather than standing on the floor (lost souls in ceilings, hanging decorations). |
| `MF_NOGRAVITY` | `0x00000200` | Gravity is not applied each tic; used for flying/floating objects that maintain their height. |
| `MF_DROPOFF` | `0x00000400` | Allowed to fall off ledges; monsters without this flag will not walk off high steps. |
| `MF_PICKUP` | `0x00000800` | Will pick up special items when walked over (player objects). |
| `MF_NOCLIP` | `0x00001000` | Ignores all clipping against walls and other objects (cheat code `IDCLIP`). |
| `MF_SLIDE` | `0x00002000` | Tracks sliding along walls (set on the player). |
| `MF_FLOAT` | `0x00004000` | Active floater; can move to any height. Used by cacodemons, pain elementals. No gravity applied. |
| `MF_TELEPORT` | `0x00008000` | Do not cross lines or check heights during teleportation. |
| `MF_MISSILE` | `0x00010000` | Projectile: does not hit same species, explodes on contact. Player and monster missiles alike. |
| `MF_DROPPED` | `0x00020000` | Dropped by a dying monster (e.g., clip from a former human) rather than placed by the map editor. Dropped items do not go into the deathmatch respawn queue. |
| `MF_SHADOW` | `0x00040000` | Fuzzy/shadow draw mode (spectres, partial-invisibility powerup). Attacks against this object have reduced accuracy. |
| `MF_NOBLOOD` | `0x00080000` | Hit effect is a puff, not a blood splat (barrels, furniture, non-organic objects). |
| `MF_CORPSE` | `0x00100000` | Dead body; will slide down steps all the way rather than stopping halfway. |
| `MF_INFLOAT` | `0x00200000` | Currently floating toward a target height; suppresses the auto-float-to-target logic to prevent oscillation. |
| `MF_COUNTKILL` | `0x00400000` | Killing this monster counts toward the intermission kill total. |
| `MF_COUNTITEM` | `0x00800000` | Picking up this item counts toward the intermission item total. |
| `MF_SKULLFLY` | `0x01000000` | Special flag for the Lost Soul in flight; neither a cacodemon-style floater nor a missile. Causes it to bounce off ceilings and floors. |
| `MF_NOTDMATCH` | `0x02000000` | Not spawned in deathmatch mode (keycards, skull keys, etc.). |
| `MF_TRANSLATION` | `0x0C000000` | Two-bit field (bits 26-27) selecting which colour translation table to use for multiplayer player sprites. |
| `MF_TRANSSHIFT` | `26` | Shift amount for extracting the translation index from the flags field. |

---

### `mobj_t` (struct)

Defined as `struct mobj_s` with `typedef mobj_t`. The universal game-object structure. Every dynamic entity in the world — players, monsters, projectiles, pickups, decorations, effects — is an instance of this struct.

The struct is laid out so that the `thinker_t` is the **first member**. This allows a pointer to an `mobj_t` to be cast directly to a `thinker_t*` and vice versa, which is fundamental to how the thinker list works.

| Type | Field | Purpose |
|------|-------|---------|
| `thinker_t` | `thinker` | Doubly-linked list node used by the thinker system (`p_tick.c`). Contains the per-tic callback function pointer. Must be first. |
| `fixed_t` | `x` | World X position (fixed-point). Origin is at the bottom-centre of the sprite. |
| `fixed_t` | `y` | World Y position (fixed-point). |
| `fixed_t` | `z` | World Z position (fixed-point). Equal to `floorz` when standing on the floor. |
| `struct mobj_s*` | `snext` | Next mobj in the owning sector's linked list. Used by the renderer to enumerate things in a sector. |
| `struct mobj_s*` | `sprev` | Previous mobj in the sector list. |
| `angle_t` | `angle` | Facing direction as a Binary Angle Measurement (BAM). Determines which sprite rotation is displayed. |
| `spritenum_t` | `sprite` | Current sprite graphic set index (copied from the current `state_t`). |
| `int` | `frame` | Current frame within the sprite set. May have `FF_FULLBRIGHT` (0x8000) OR'd in to render at full brightness. |
| `struct mobj_s*` | `bnext` | Next mobj in the blockmap block's linked list. Used by the collision system. |
| `struct mobj_s*` | `bprev` | Previous mobj in the blockmap block list. |
| `struct subsector_s*` | `subsector` | The subsector that contains this mobj's origin. Set by `P_SetThingPosition`. Also used by the sound system for stereo positioning. |
| `fixed_t` | `floorz` | The highest floor height across all contacted sectors. The lowest z the mobj can rest at. |
| `fixed_t` | `ceilingz` | The lowest ceiling height across all contacted sectors. |
| `fixed_t` | `radius` | Collision cylinder radius (fixed-point). |
| `fixed_t` | `height` | Collision cylinder height (fixed-point). |
| `fixed_t` | `momx` | X-axis momentum applied each tic. |
| `fixed_t` | `momy` | Y-axis momentum applied each tic. |
| `fixed_t` | `momz` | Z-axis momentum applied each tic (affected by gravity). |
| `int` | `validcount` | Stamp compared to the global `validcount` to avoid processing the same object twice in a single traversal (e.g., blockmap iteration). |
| `mobjtype_t` | `type` | Index into `mobjinfo[]`; identifies what kind of object this is. |
| `mobjinfo_t*` | `info` | Cached pointer to `&mobjinfo[mobj->type]` for fast access to type constants. |
| `int` | `tics` | Tic counter for the current animation state. Decremented each tic; when it reaches zero the state advances. A value of `-1` means the object is in a permanent (infinite) state. |
| `state_t*` | `state` | Pointer to the current animation/behaviour state in the `states[]` array. |
| `int` | `flags` | Behaviour flags; a bitmask of `mobjflag_t` values. |
| `int` | `health` | Current hit points. When <= 0, the death sequence is triggered. |
| `int` | `movedir` | Monster movement direction index (0-7, corresponding to compass directions) used by the AI. |
| `int` | `movecount` | Number of tics remaining before the AI selects a new movement direction. Also reused as a dead-monster respawn timer in nightmare mode. |
| `struct mobj_s*` | `target` | The mobj this object is chasing or attacking. For missiles, this is the mobj that fired them (the originator). |
| `int` | `reactiontime` | Countdown preventing attack immediately after spawn or teleport. On Nightmare skill this is always 0 at spawn time. |
| `int` | `threshold` | When > 0, the monster will continue chasing `target` even if shot by someone else. Decrements each tic. |
| `struct player_s*` | `player` | Non-NULL only for `MT_PLAYER` objects. Pointer to the player data structure. |
| `int` | `lastlook` | Index of the last player this monster looked for. Used by the wake-up code to cycle through all players fairly. |
| `mapthing_t` | `spawnpoint` | Copy of the original map-thing record. Used by nightmare respawn to recreate the monster and by the deathmatch item-respawn queue. |
| `struct mobj_s*` | `tracer` | Secondary tracking target used by homing missiles (Revenant tracer rockets in Doom 2). |

---

## Notes on the Architecture (from source comments)

The header contains detailed design notes that are worth summarising:

- **Coordinate origin:** The XY position is the centre of the collision cylinder; Z is the bottom. This is the foot position for bipedal sprites.
- **Sector links** (`snext`/`sprev`): Used only by the renderer to walk objects in a sector. The simulation ignores these.
- **Blockmap links** (`bnext`/`bprev`): Used by the physics/collision simulation. Objects with `MF_NOBLOCKMAP` cannot be collided into (though they can still collide with others as the instigator).
- **Validity rule:** An `mobj_t` is "valid" if its `subsector` pointer is correct for its XY position and it is in either the sector list or has `MF_NOSECTOR`, and in either the blockmap or has `MF_NOBLOCKMAP`. The only functions allowed to touch the link pointers are `P_SetThingPosition` and `P_UnsetThingPosition`.
- **Do not change `MF_NO*` flags** while a thing is valid — this would corrupt the spatial index.

---

## Dependencies

| Header | What is used |
|--------|-------------|
| `tables.h` | `angle_t`, `FINEANGLES`, BAM constants |
| `m_fixed.h` | `fixed_t` type and fixed-point constants |
| `d_think.h` | `thinker_t` (embedded as first member of `mobj_t`) |
| `doomdata.h` | `mapthing_t` (used for `spawnpoint` and spawn parameters) |
| `info.h` | `state_t`, `mobjinfo_t`, `mobjtype_t`, `spritenum_t`, `statenum_t` |
