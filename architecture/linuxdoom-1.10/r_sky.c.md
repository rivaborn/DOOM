# File Overview

**File:** `linuxdoom-1.10/r_sky.c`
**Module:** Renderer - Sky Mapping

This file implements sky rendering initialization for the DOOM engine. The sky is a special case in DOOM's renderer: rather than projecting geometry, upper walls whose ceiling texture matches the magic flat `F_SKY1` are rendered as sky columns using a texture-mapped panorama.

The sky texture wraps around the full 360-degree view. The engine maps 1024 virtual columns to 360 degrees of view, meaning a default 256-column sky texture repeats approximately four times around the full circle. The mapping is driven by the `ANGLETOSKYSHIFT` constant (defined in `r_sky.h`), which shifts a screen-space angle down 22 bits to derive a texture column index. This means the texture coordinate is sampled from the upper 10 bits of the 32-bit angle, giving the 1024-column virtual panorama.

The actual sky column drawing is performed by `r_segs.c` and `r_plane.c`, which consult the globals defined here (`skytexture`, `skytexturemid`) to render sky-tagged walls.

---

## Global Variables

| Type  | Name            | Purpose |
|-------|-----------------|---------|
| `int` | `skyflatnum`    | WAD flat number that marks a ceiling as sky (the flat named `F_SKY1`). Sectors whose ceiling `picnum` equals this value are rendered as open sky. Defined here but commented out in `R_InitSkyMap`; set elsewhere during level load via `R_FlatNumForName`. |
| `int` | `skytexture`    | WAD texture number of the sky texture to draw. Set during level load (e.g. in `P_LoadSectors`) based on the episode/map. Used by `r_segs.c` to select which texture to sample when rendering sky columns. |
| `int` | `skytexturemid` | The texture mid-point (in fixed-point `FRACUNIT` units) used for vertical sky alignment. Initialized to `100 * FRACUNIT`, placing the sky's vertical centre 100 pixels from the top of a 200-pixel-tall reference view. Controls how the sky texture is vertically positioned relative to the horizon. |

---

## Functions

### `R_InitSkyMap`

```c
void R_InitSkyMap(void)
```

**Purpose:** Initializes sky mapping state. Called whenever the view size changes (i.e., when the player resizes the screen via the `+`/`-` keys), ensuring the sky vertical center remains correct for the current viewport.

**Parameters:** None.

**Return value:** None (`void`).

**Key logic:**

- Sets `skytexturemid = 100 * FRACUNIT`. The value 100 corresponds to the vertical midpoint of the classic 200-pixel-tall DOOM view. This anchors the sky horizon at the correct screen position.
- A commented-out call to `R_FlatNumForName(SKYFLATNAME)` shows the original intent to also initialize `skyflatnum` here; in practice `skyflatnum` is set during `R_InitData` / level loading.

**Notes:** This function is minimal. Because the sky does not have perspective-correct vertical scaling (it is drawn with a fixed vertical scale), the `skytexturemid` initialization here is sufficient for all view sizes; the horizontal angle mapping via `ANGLETOSKYSHIFT` handles view direction automatically.

---

## Data Structures

None defined in this file. The file relies on types from `m_fixed.h` (`fixed_t`, `FRACUNIT`) and the module's own header.

---

## Dependencies

| File | Reason |
|------|--------|
| `m_fixed.h` | Provides `fixed_t`, `FRACUNIT` for `skytexturemid` initialization |
| `r_data.h`  | Provides flat/texture lookup utilities; listed as "needed for Flat retrieval" in comments, though the `R_FlatNumForName` call is currently commented out |
| `r_sky.h`   | Own public interface; declares `skytexture`, `skytexturemid`, `R_InitSkyMap`, `SKYFLATNAME`, `ANGLETOSKYSHIFT` |
