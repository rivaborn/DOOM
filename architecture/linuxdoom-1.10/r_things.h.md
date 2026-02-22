# File Overview

**File:** `linuxdoom-1.10/r_things.h`
**Module:** Renderer - Sprite / Thing Rendering (Public Interface)

This header is the public interface for `r_things.c`, exporting the constants, globals, and function declarations needed by other renderer modules to interact with the sprite rendering subsystem. It is included by `r_local.h`, making its declarations available throughout the renderer.

Key consumers:
- `r_main.c` - calls `R_InitSprites` at startup and `R_ClearSprites` at the start of each frame.
- `r_bsp.c` - calls `R_AddSprites` for each visible sector during BSP traversal.
- `r_plane.c` and `r_segs.c` - use `mfloorclip`, `mceilingclip`, `spryscale`, `sprtopscreen`, and `R_DrawMaskedColumn` for masked mid-texture rendering.

---

## Constants

### `MAXVISSPRITES`

```c
#define MAXVISSPRITES  128
```

Maximum number of sprites that can be visible in a single rendered frame. If more sprites are projected than this limit, `R_NewVisSprite` returns a pointer to the dummy `overflowsprite` sink and subsequent sprites are silently discarded. At 128 sprites, this limit is rarely hit in normal gameplay but can be exceeded in heavily populated areas or with source port modifications.

---

## Exported Globals

### Vissprite pool

| Type           | Name              | Defined in    | Purpose |
|----------------|-------------------|---------------|---------|
| `vissprite_t`  | `vissprites[MAXVISSPRITES]` | `r_things.c` | Fixed pool of per-frame visible sprite records. Each entry is populated by `R_ProjectSprite`. |
| `vissprite_t*` | `vissprite_p`     | `r_things.c` | Pointer one past the last used entry in `vissprites[]`. Difference `vissprite_p - vissprites` gives the sprite count for the current frame. |
| `vissprite_t`  | `vsprsortedhead`  | `r_things.c` | Doubly-linked list sentinel. After `R_SortVisSprites`, traversal from `vsprsortedhead.next` to `vsprsortedhead` (exclusive) visits sprites farthest-to-nearest. |

### Clipping arrays

| Type    | Name                 | Defined in   | Purpose |
|---------|----------------------|--------------|---------|
| `short` | `negonearray[SCREENWIDTH]` | `r_things.c` | All-(-1) array. Used as `mceilingclip` for player weapon sprites (no ceiling clip). Also used to initialize clipping in `R_DrawSprite`. |
| `short` | `screenheightarray[SCREENWIDTH]` | `r_things.c` | All-`viewheight` array. Used as `mfloorclip` for player weapon sprites (no floor clip). |

### Masked-column rendering state

These variables are set before calling `R_DrawMaskedColumn` and are shared with `r_segs.c` for masked mid-texture rendering.

| Type      | Name           | Defined in   | Purpose |
|-----------|----------------|--------------|---------|
| `short*`  | `mfloorclip`   | `r_things.c` | Per-column array of bottom clip Y values. `R_DrawMaskedColumn` clips each post's bottom to `mfloorclip[dc_x] - 1`. |
| `short*`  | `mceilingclip` | `r_things.c` | Per-column array of top clip Y values. `R_DrawMaskedColumn` clips each post's top to `mceilingclip[dc_x] + 1`. |
| `fixed_t` | `spryscale`    | `r_things.c` | Vertical scale (pixels per world unit) for the column currently being drawn by `R_DrawMaskedColumn`. |
| `fixed_t` | `sprtopscreen` | `r_things.c` | Fixed-point Y coordinate of the top of the current sprite texture column on screen. Pre-computed by `R_DrawVisSprite`. |

### Player weapon sprite scaling

| Type      | Name            | Defined in   | Purpose |
|-----------|-----------------|--------------|---------|
| `fixed_t` | `pspritescale`  | `r_things.c` | Horizontal scale for HUD (player weapon) sprites. Set in `R_ExecuteSetViewSize`. |
| `fixed_t` | `pspriteiscale` | `r_things.c` | Inverse of `pspritescale`. Used as `dc_iscale` (texture step per screen column) for weapon sprite columns. |

---

## Functions

### `R_DrawMaskedColumn`

```c
void R_DrawMaskedColumn(column_t* column);
```

Renders one masked patch column by iterating its post list. Used both for world sprites (via `R_DrawVisSprite`) and for masked mid-textures on two-sided linedefs (via `R_RenderMaskedSegRange` in `r_segs.c`). Requires `mfloorclip`, `mceilingclip`, `spryscale`, `sprtopscreen`, `dc_x`, and `dc_texturemid` to be set by the caller.

---

### `R_SortVisSprites`

```c
void R_SortVisSprites(void);
```

Sorts the vissprites collected this frame into the `vsprsortedhead` doubly-linked list, ordered from smallest scale (farthest) to largest (closest). Called once per frame from `R_DrawMasked`.

---

### `R_AddSprites`

```c
void R_AddSprites(sector_t* sec);
```

Projects all things in the given sector into vissprites. Called by the BSP traversal code (`R_Subsector` in `r_bsp.c`) for each visible subsector's parent sector. Guards against processing the same sector twice per frame via `validcount`.

---

### `R_AddPSprites` (declared but not defined in r_things.c)

```c
void R_AddPSprites(void);
```

Declared here but the implementation present in `r_things.c` is named `R_DrawPlayerSprites`. This declaration appears to be a naming inconsistency / earlier API name retained in the header.

---

### `R_DrawSprites` (declared but not implemented under this name)

```c
void R_DrawSprites(void);
```

Declared here but the actual entry point in `r_things.c` is `R_DrawMasked`. This is another header/implementation naming mismatch from the original source.

---

### `R_InitSprites`

```c
void R_InitSprites(char** namelist);
```

One-time startup initialization. Builds the sprite definition tables from WAD lump names and initializes the `negonearray` clipping buffer.

---

### `R_ClearSprites`

```c
void R_ClearSprites(void);
```

Per-frame reset: sets `vissprite_p = vissprites` to reclaim the vissprite pool.

---

### `R_DrawMasked`

```c
void R_DrawMasked(void);
```

Main entry point for the masked rendering pass. Called once per frame from `R_RenderPlayerView` after solid geometry has been drawn. Sorts vissprites, draws them back-to-front, renders masked mid-textures, then draws player weapon sprites.

---

### `R_ClipVisSprite` (declared but not implemented in r_things.c)

```c
void R_ClipVisSprite(vissprite_t* vis, int xl, int xh);
```

Declared in this header but has no corresponding implementation in `r_things.c` in the released source. The clipping logic it would have provided was folded directly into `R_DrawSprite`.

---

## Dependencies

| File | Reason |
|------|--------|
| *(none directly)* | This header does not include other headers itself. It relies on types (`vissprite_t`, `sector_t`, `column_t`, `fixed_t`, `short`, `SCREENWIDTH`) that are available via `r_local.h` in any translation unit that includes this header through that aggregator. |

Include guard: `__R_THINGS__`
