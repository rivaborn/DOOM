# File Overview

**File:** `linuxdoom-1.10/r_sky.h`
**Module:** Renderer - Sky Mapping (Public Interface)

This header exposes the sky rendering constants and externals that other renderer modules need. The sky in DOOM is not rendered as a true geometric surface; instead, any wall or plane whose ceiling texture equals the flat `F_SKY1` is drawn using a horizontal panorama texture that scrolls with the player's viewing angle. This header defines the two key constants that make that mapping work and exports the runtime state variables maintained by `r_sky.c`.

Consumers of this header:
- `r_segs.c` - tests `skyflatnum` when processing upper textures and uses `skytexture` / `skytexturemid` to draw sky columns.
- `r_plane.c` - similarly skips floor/ceiling visplane fill when the flat is `F_SKY1`.
- `p_setup.c` - sets `skytexture` to the correct episode sky during map loading.

---

## Global Variables (Exported Externals)

| Type  | Name            | Defined in   | Purpose |
|-------|-----------------|-------------|---------|
| `int` | `skytexture`    | `r_sky.c`   | WAD texture number of the current sky panorama. Set each time a new map is loaded to reflect the episode-appropriate sky (e.g., `SKY1`, `SKY2`, `SKY3`). |
| `int` | `skytexturemid` | `r_sky.c`   | Fixed-point vertical center coordinate for sky texture sampling. Initialized to `100 * FRACUNIT`; stays constant regardless of view size because sky has no perspective vertical warp. |

The variable `skyflatnum` is defined in `r_sky.c` but is **not** exported from this header; it is instead made available to the rest of the renderer through `r_state.h` or set internally.

---

## Macros / Constants

### `SKYFLATNAME`

```c
#define SKYFLATNAME  "F_SKY1"
```

The WAD name of the special ceiling flat that triggers sky rendering. When the renderer encounters a sector whose ceiling picture number equals `R_FlatNumForName("F_SKY1")`, it skips normal plane drawing and instead renders sky columns.

### `ANGLETOSKYSHIFT`

```c
#define ANGLETOSKYSHIFT  22
```

The bit-shift applied to a screen-column view angle (a 32-bit `angle_t`) to derive a sky texture column index. Shifting a full 32-bit angle right by 22 bits yields a 10-bit index in the range `[0, 1023]`, mapping the full 360-degree panorama to 1024 virtual texture columns. Because the default sky texture is 256 pixels wide, the panorama repeats four times around a full rotation. The formula used in `r_segs.c` is:

```c
// angle is the view angle for the current column
int skytexturecol = angle >> ANGLETOSKYSHIFT;
```

---

## Functions

### `R_InitSkyMap`

```c
void R_InitSkyMap(void);
```

**Purpose:** Re-initializes sky mapping state after a view-size change. Sets `skytexturemid` to `100 * FRACUNIT`.

**Parameters:** None.

**Return value:** None (`void`).

Implemented in `r_sky.c`. Called from `R_ExecuteSetViewSize` in `r_main.c` whenever the player adjusts the screen size.

---

## Dependencies

| File | Reason |
|------|--------|
| *(none directly)* | The header itself includes no other headers. Callers typically already include `m_fixed.h` and `r_data.h` for the types used in the implementation. |

Include guard: `__R_SKY__`
