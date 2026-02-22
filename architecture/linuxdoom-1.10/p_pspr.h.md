# File Overview

**Source file:** `linuxdoom-1.10/p_pspr.h`

`p_pspr.h` declares the types and constants needed to represent **player weapon sprites** (psprites) — the first-person weapon overlay drawn at the bottom of the player's screen. This header is included by `p_pspr.c` (the implementation), `d_player.h` (which embeds `pspdef_t` into the player structure), and any module that needs to inspect or manipulate the weapon animation state.

The header is intentionally minimal: it defines one enum, one struct, and two frame-flag macros. The full implementation is in `p_pspr.c`.

---

## Preprocessor Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `FF_FULLBRIGHT` | `0x8000` | Bit flag OR'd into a sprite frame number to indicate the frame should be rendered at maximum brightness regardless of sector lighting. Used for muzzle flashes, fire effects, and light-emitting objects. |
| `FF_FRAMEMASK` | `0x7FFF` | Mask to extract the raw frame index from a value that may have `FF_FULLBRIGHT` set. Apply this to `mobj->frame` or `pspdef_t` frame data to get the actual sprite frame number. |

Note: Although these frame flags are defined in `p_pspr.h`, they apply to all sprite animation frames in the engine — including world-object (`mobj_t`) frames — not just psprites. The renderer uses `FF_FRAMEMASK` and `FF_FULLBRIGHT` when drawing both weapon sprites and world objects.

---

## Data Structures

### `psprnum_t` (enum)

Identifies which overlay psprite slot is being addressed.

```c
typedef enum
{
    ps_weapon,    // 0 -- the main weapon body sprite
    ps_flash,     // 1 -- the muzzle flash overlay sprite
    NUMPSPRITES   // 2 -- total number of psprite slots
} psprnum_t;
```

| Value | Index | Meaning |
|-------|-------|---------|
| `ps_weapon` | 0 | The primary weapon sprite. Draws the gun body, animates through raise/lower/ready/attack states. Its screen position is the authoritative one; the flash copies it every tic. |
| `ps_flash` | 1 | The muzzle flash overlay. Drawn on top of `ps_weapon` at the same screen position. Activated by `A_GunFlash` during firing states; immediately deactivated after a few tics. |
| `NUMPSPRITES` | 2 | Sentinel value used to size the `psprites[NUMPSPRITES]` array in `player_t`. |

---

### `pspdef_t` (struct)

Per-slot state record for a single overlay psprite. The `player_t` structure contains an array of `NUMPSPRITES` of these.

```c
typedef struct
{
    state_t*  state;   // current animation state; NULL means slot is inactive
    int       tics;    // tics remaining in this state
    fixed_t   sx;      // screen-space X position (320-wide coordinate space)
    fixed_t   sy;      // screen-space Y position (200-high coordinate space)
} pspdef_t;
```

| Type | Field | Purpose |
|------|-------|---------|
| `state_t*` | `state` | Pointer into the global `states[]` array. When `NULL`, this psprite slot is inactive and nothing is drawn for it. The main game loop (`P_MovePsprites`) checks this each tic. |
| `int` | `tics` | Countdown timer for the current state. Decremented each tic by `P_MovePsprites`. When it reaches zero, `P_SetPsprite` is called to advance to `state->nextstate`. A value of `-1` indicates a permanent state that never advances. |
| `fixed_t` | `sx` | Horizontal screen position in fixed-point, using a 320-pixel-wide virtual screen coordinate system. `FRACUNIT` equals 1 pixel. The default resting position is approximately `FRACUNIT` (1 pixel from the left edge as set by the bob calculation). |
| `fixed_t` | `sy` | Vertical screen position. `WEAPONTOP` (32*FRACUNIT) is the fully-raised position; `WEAPONBOTTOM` (128*FRACUNIT) is fully off-screen below. The weapon raise/lower animations work by decrementing/incrementing this value until the threshold is crossed. |

**Screen coordinate conventions:**
- The psprite coordinate system is the 320x200 virtual screen used by the original DOS renderer.
- Origin (0, 0) is top-left.
- `sx` and `sy` are in fixed-point (`fixed_t`), so one full pixel is `FRACUNIT = 1<<16`.
- The bob animation offsets are added to these values in `A_WeaponReady`.
- The `ps_flash` slot's `sx`/`sy` are copied from `ps_weapon` each tic in `P_MovePsprites` so both layers stay aligned.

---

## Dependencies

| Header | What is used |
|--------|-------------|
| `m_fixed.h` | `fixed_t` type used for `sx`, `sy` |
| `tables.h` | `angle_t` (transitively needed by `info.h`) |
| `info.h` | `state_t` (the type of the `state` pointer field) |
