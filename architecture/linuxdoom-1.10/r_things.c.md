# File Overview

**File:** `linuxdoom-1.10/r_things.c`
**Module:** Renderer - Sprite / Thing Rendering

This file implements the full sprite rendering pipeline for DOOM. "Things" are all moveable objects in the game world (monsters, pickups, projectiles, decorations, the player's weapon) and are drawn as 2-D billboard sprites that always face the camera.

The rendering pipeline has four major phases:

1. **Sprite definition loading** (`R_InitSpriteDefs`, `R_InstallSpriteLump`) - Runs at startup. Parses WAD lump names to build `spritedef_t` / `spriteframe_t` tables that map each sprite's animation frame and view rotation to the correct patch lump.

2. **Projection** (`R_ProjectSprite`, `R_AddSprites`) - Runs during BSP traversal each frame. For each visible `mobj_t`, transforms its 3-D world position into screen space, selects the correct rotation frame, and fills a `vissprite_t` record into the `vissprites[]` array.

3. **Sorting** (`R_SortVisSprites`) - After BSP traversal is complete, sorts the `vissprites[]` array from farthest to nearest by projected scale (larger scale = closer). DOOM uses an insertion-sort-by-selection into a doubly-linked list rather than a standard sort.

4. **Drawing** (`R_DrawMasked`, `R_DrawSprite`, `R_DrawVisSprite`, `R_DrawMaskedColumn`) - Draws each sprite back-to-front, clipping against the `drawseg_t` records left by the wall renderer to correctly hide sprites behind walls. Also draws player weapon sprites (`R_DrawPlayerSprites`, `R_DrawPSprite`) on top of everything else.

---

## Global Variables

### Sprite scaling for player weapon (psprite)

| Type      | Name            | Purpose |
|-----------|-----------------|---------|
| `fixed_t` | `pspritescale`  | Horizontal scale factor for player weapon (HUD) sprites. Computed as `projection / BASEYCENTER` (effectively `viewwidth / SCREENWIDTH`). Not perspective-dependent since weapon sprites are drawn in screen space. |
| `fixed_t` | `pspriteiscale` | Inverse of `pspritescale` (`FRACUNIT / pspritescale`). Used as the per-column texture step when traversing a psprite's columns. |

### Lighting

| Type             | Name           | Purpose |
|------------------|----------------|---------|
| `lighttable_t**` | `spritelights` | Pointer into the `scalelight[][]` array, set per-sector during `R_AddSprites` or `R_DrawPlayerSprites`. Points to the row of 48 light-scale entries for the current sector's light level. |

### Clipping arrays

| Type    | Name                | Purpose |
|---------|---------------------|---------|
| `short` | `negonearray[SCREENWIDTH]` | Pre-filled with -1 for every column. Used as `mceilingclip` for player weapon sprites, meaning "no ceiling clip, draw from the top". |
| `short` | `screenheightarray[SCREENWIDTH]` | Pre-filled with `viewheight` for every column (filled lazily; actually set to SCREENWIDTH constant -1 rows). Used as `mfloorclip` for player weapon sprites, meaning "no floor clip, draw to the bottom". |

### Sprite definition tables

| Type           | Name        | Purpose |
|----------------|-------------|---------|
| `spritedef_t*` | `sprites`   | Heap-allocated array of `numsprites` sprite definitions, one per sprite name (e.g., TROO, POSS, PLAY). Built by `R_InitSpriteDefs`. |
| `int`          | `numsprites`| Count of entries in `sprites[]`. |
| `spriteframe_t`| `sprtemp[29]` | Temporary workspace used during `R_InitSpriteDefs` to accumulate frame/rotation data for one sprite before copying to `sprites[i].spriteframes`. |
| `int`          | `maxframe`  | Highest frame index seen during WAD scan for current sprite, used to size the allocation. |
| `char*`        | `spritename`| Pointer to the current sprite's 4-character name (points into `namelist`). Used only for error messages during init. |

### Vissprite pool

| Type          | Name              | Purpose |
|---------------|-------------------|---------|
| `vissprite_t` | `vissprites[MAXVISSPRITES]` | Fixed-size pool (128 entries) of visible sprite records for the current frame. |
| `vissprite_t*`| `vissprite_p`     | Next-free pointer into `vissprites[]`. Reset to `vissprites` at frame start by `R_ClearSprites`. |
| `int`         | `newvissprite`    | Unused in the final code (vestigial). |
| `vissprite_t` | `overflowsprite`  | Dummy sink record returned by `R_NewVisSprite` when the pool is full, preventing buffer overruns at the cost of silently discarding the sprite. |
| `vissprite_t` | `vsprsortedhead`  | Sentinel node for the sorted doubly-linked list produced by `R_SortVisSprites`. Sprites are drawn front-to-back via `vsprsortedhead.next`. |

### Masked column rendering state

| Type      | Name            | Purpose |
|-----------|-----------------|---------|
| `short*`  | `mfloorclip`    | Per-column floor clip array passed to `R_DrawMaskedColumn`. Column drawing stops at `mfloorclip[dc_x] - 1`. |
| `short*`  | `mceilingclip`  | Per-column ceiling clip array. Column drawing starts at `mceilingclip[dc_x] + 1`. |
| `fixed_t` | `spryscale`     | Current sprite's projected vertical scale (pixels-per-unit) used to convert texture row deltas to screen pixels in `R_DrawMaskedColumn`. |
| `fixed_t` | `sprtopscreen`  | Screen Y coordinate (in fixed-point) of the top of the current sprite's texture at scale `spryscale`. Pre-computed as `centeryfrac - FixedMul(dc_texturemid, spryscale)`. |

---

## Functions

### `R_InstallSpriteLump`

```c
void R_InstallSpriteLump(int lump, unsigned frame, unsigned rotation, boolean flipped)
```

**Purpose:** Records one sprite patch lump into the `sprtemp[]` working frame table during startup initialization.

**Parameters:**
- `lump` - Absolute WAD lump index of the sprite patch.
- `frame` - Frame index (0 = 'A', 1 = 'B', ...).
- `rotation` - Rotation index: 0 means "use for all rotations"; 1-8 means a specific view angle (stored as 0-7 after subtracting 1).
- `flipped` - If `true`, the patch should be rendered mirrored horizontally.

**Return value:** None.

**Key logic:**
- If `rotation == 0`, fills all 8 slots in `sprtemp[frame].lump[]` and `sprtemp[frame].flip[]` with the same lump and flip flag, and sets `sprtemp[frame].rotate = false`.
- If `rotation != 0`, records the lump into the specific slot `sprtemp[frame].lump[rotation-1]` and sets `rotate = true`.
- Errors (via `I_Error`) on duplicate assignments (a frame cannot have both a rotation-0 and rotation-specific patches, and no slot can be written twice).
- `lump` is stored relative to `firstspritelump` so that `spriteframe_t.lump[]` values are compact offsets.

---

### `R_InitSpriteDefs`

```c
void R_InitSpriteDefs(char** namelist)
```

**Purpose:** Builds the complete `sprites[]` array at program startup by scanning all WAD sprite lumps.

**Parameters:**
- `namelist` - NULL-terminated array of 4-character sprite name strings (e.g., `{"TROO", "POSS", NULL}`).

**Return value:** None.

**Key logic:**
1. Counts entries in `namelist` to set `numsprites`; allocates `sprites[]` via `Z_Malloc`.
2. For each sprite name, resets `sprtemp` to -1 and scans every lump in `[firstspritelump, lastspritelump]`.
3. For each matching lump (first 4 chars of name match), extracts frame (`name[4] - 'A'`) and rotation (`name[5] - '0'`) and calls `R_InstallSpriteLump`. If the lump name has a second frame/rotation pair (`name[6]`/`name[7]`), that is also installed (with `flipped = true`), allowing one lump to serve two view angles.
4. If `modifiedgame` (a PWAD is loaded), the actual patch is fetched via `W_GetNumForName` to allow overrides.
5. Validates completeness: a frame with `rotate == true` must have all 8 rotation slots filled.
6. Copies `sprtemp[]` into a `Z_Malloc`-allocated `spriteframes` array for each sprite.

---

### `R_InitSprites`

```c
void R_InitSprites(char** namelist)
```

**Purpose:** Top-level sprite initialization called once at program startup.

**Parameters:**
- `namelist` - NULL-terminated list of sprite names to register (passed through to `R_InitSpriteDefs`).

**Return value:** None.

**Key logic:**
- Pre-fills `negonearray[]` with -1 across all `SCREENWIDTH` columns.
- Calls `R_InitSpriteDefs(namelist)`.

---

### `R_ClearSprites`

```c
void R_ClearSprites(void)
```

**Purpose:** Resets the vissprite pool at the start of each rendered frame.

**Parameters:** None.

**Return value:** None.

**Key logic:** Sets `vissprite_p = vissprites`, making all pool slots available without zeroing them.

---

### `R_NewVisSprite`

```c
vissprite_t* R_NewVisSprite(void)
```

**Purpose:** Allocates the next vissprite from the pool.

**Parameters:** None.

**Return value:** Pointer to the newly allocated `vissprite_t`, or `&overflowsprite` if the pool is exhausted (silently drops the sprite).

**Key logic:** Checks if `vissprite_p == &vissprites[MAXVISSPRITES]`; if so returns the overflow sink. Otherwise increments `vissprite_p` and returns the previous value.

---

### `R_DrawMaskedColumn`

```c
void R_DrawMaskedColumn(column_t* column)
```

**Purpose:** Renders one column of a masked (transparent) patch - used for both world sprites and player weapon sprites.

**Parameters:**
- `column` - Pointer to the first `post_t` in the column's post list. A column is a sequence of opaque runs (posts) separated by transparent gaps.

**Return value:** None.

**Key logic:**
- Iterates over posts until `column->topdelta == 0xff` (sentinel).
- For each post, converts `topdelta` and `length` to screen coordinates using `sprtopscreen` and `spryscale`.
- Clamps the column's top (`dc_yl`) against `mceilingclip[dc_x]` and bottom (`dc_yh`) against `mfloorclip[dc_x]`.
- Sets `dc_source` to the pixel data bytes (3 bytes into the post header) and `dc_texturemid` accordingly, then calls the current `colfunc()` function pointer.
- Advances `column` pointer past the post: `(byte*)column + column->length + 4` (4-byte post header: topdelta, length, 2 padding bytes).

---

### `R_DrawVisSprite`

```c
void R_DrawVisSprite(vissprite_t* vis, int x1, int x2)
```

**Purpose:** Renders a fully clipped visible sprite to the screen by iterating its patch columns.

**Parameters:**
- `vis` - The vissprite to draw (contains position, scale, colormap, patch index).
- `x1`, `x2` - Horizontal screen column range to draw.

**Return value:** None.

**Key logic:**
1. Loads the patch from WAD cache: `W_CacheLumpNum(vis->patch + firstspritelump, PU_CACHE)`.
2. Selects the correct `colfunc`:
   - `NULL` colormap → shadow/fuzz effect: `colfunc = fuzzcolfunc`.
   - `MF_TRANSLATION` flag set → color translation: `colfunc = R_DrawTranslatedColumn`, with `dc_translation` pointed at the appropriate shirt-color table.
   - Normal case: `dc_colormap = vis->colormap`.
3. Sets `dc_iscale`, `dc_texturemid`, `spryscale`, `sprtopscreen`.
4. Iterates `dc_x` from `x1` to `x2`, computing `texturecolumn = frac >> FRACBITS` (where `frac` advances by `vis->xiscale` per column), then calls `R_DrawMaskedColumn`.
5. Restores `colfunc = basecolfunc` after drawing.

---

### `R_ProjectSprite`

```c
void R_ProjectSprite(mobj_t* thing)
```

**Purpose:** Projects a world-space `mobj_t` into a `vissprite_t` for later drawing, if the object is potentially visible.

**Parameters:**
- `thing` - The map object to project.

**Return value:** None.

**Key logic:**

**Transformation:**
```
tr_x = thing->x - viewx
tr_y = thing->y - viewy
tz   = FixedMul(tr_x, viewcos) - FixedMul(tr_y, viewsin)   // depth
tx   = -(FixedMul(tr_y, viewcos) + FixedMul(tr_x, viewsin)) // lateral offset
```
- Rejects if `tz < MINZ` (4 units, behind or too close to view plane).
- Rejects if `abs(tx) > tz * 4` (more than ~76 degrees off center, outside the view cone).

**Scale:** `xscale = projection / tz` (perspective divide).

**Rotation selection:** If the sprite frame has rotations, computes the player-relative view angle:
```c
rot = (ang - thing->angle + ANG45/2 * 9) >> 29;
```
This maps the 8 cardinal/diagonal view directions to rotation indices 0-7. The `* 9` bias centers the selection on the nearest direction. If no rotations, uses lump[0].

**Screen edge computation:**
```c
x1 = (centerxfrac + FixedMul(tx - spriteoffset[lump], xscale)) >> FRACBITS
x2 = (centerxfrac + FixedMul(tx + spritewidth[lump] - spriteoffset[lump], xscale)) >> FRACBITS - 1
```
Rejects if entirely off-screen.

**Fills vissprite:**
- `vis->gx/gy/gz/gzt` - world-space bounding box corners.
- `vis->texturemid = vis->gzt - viewz` - vertical texture coordinate of the top of the sprite relative to view height.
- `vis->startfrac` / `vis->xiscale` - handle horizontal flip: if flipped, `startfrac = spritewidth-1`, `xiscale = -iscale`.
- Adjusts `startfrac` for the case where `x1` was clamped to 0 (sprite extends left of screen).

**Lighting:**
- `MF_SHADOW` → `colormap = NULL` (fuzz/shadow effect).
- `fixedcolormap` set → use fixed colormap (e.g., ALLMAP cheat or invulnerability).
- `FF_FULLBRIGHT` frame flag → use `colormaps` (brightest map, index 0).
- Otherwise → diminished light: `spritelights[xscale >> (LIGHTSCALESHIFT - detailshift)]`, clamped to `[0, MAXLIGHTSCALE-1]`.

---

### `R_AddSprites`

```c
void R_AddSprites(sector_t* sec)
```

**Purpose:** Projects all things in a sector during BSP traversal.

**Parameters:**
- `sec` - The sector whose thing list to process.

**Return value:** None.

**Key logic:**
- Guards against double-processing the same sector in one frame using `sec->validcount == validcount`.
- Sets `spritelights` from `scalelight[lightnum]`, where `lightnum` is derived from `sec->lightlevel >> LIGHTSEGSHIFT + extralight`.
- Iterates `sec->thinglist` linked list and calls `R_ProjectSprite(thing)` for each.

---

### `R_DrawPSprite`

```c
void R_DrawPSprite(pspdef_t* psp)
```

**Purpose:** Draws one player weapon sprite (a "psprite" - player sprite). Unlike world sprites, psprites are drawn in a fixed screen-space position, not perspective-projected from a world position.

**Parameters:**
- `psp` - Player sprite definition, containing state pointer (which encodes the sprite name and frame) and screen-space `sx`/`sy` offsets.

**Return value:** None.

**Key logic:**
- Uses `pspritescale` instead of a perspective-derived scale.
- `tx = psp->sx - 160 * FRACUNIT` centers the sprite in the 320-pixel-wide reference frame.
- `vis->texturemid = (BASEYCENTER << FRACBITS) + FRACUNIT/2 - (psp->sy - spritetopoffset[lump])` places the weapon vertically based on its bob position.
- Uses a stack-allocated `vissprite_t avis` rather than a pool slot (psprites don't participate in sorting).
- Lighting: invisibility power-up → shadow (`colormap = NULL`); otherwise same logic as `R_ProjectSprite`.
- Calls `R_DrawVisSprite` directly (no clipping pass needed; `mfloorclip`/`mceilingclip` are set to `screenheightarray`/`negonearray` by the caller `R_DrawPlayerSprites`).

---

### `R_DrawPlayerSprites`

```c
void R_DrawPlayerSprites(void)
```

**Purpose:** Sets up lighting and clipping for player weapon sprites, then draws all active psprites.

**Parameters:** None.

**Return value:** None.

**Key logic:**
- Sets `spritelights` from the light level of the sector the player is standing in.
- Sets `mfloorclip = screenheightarray` and `mceilingclip = negonearray` - these are full-screen bounds so weapon sprites are never clipped by geometry.
- Iterates `viewplayer->psprites[0..NUMPSPRITES-1]` and calls `R_DrawPSprite` for each non-null state.

---

### `R_SortVisSprites`

```c
void R_SortVisSprites(void)
```

**Purpose:** Sorts the `vissprites[]` pool into a doubly-linked list ordered from farthest to nearest (ascending `scale`) for correct back-to-front painter's algorithm rendering.

**Parameters:** None.

**Return value:** None.

**Key logic:**
1. Chains all `vissprite_t` records into a doubly-linked circular list around a local sentinel `unsorted`.
2. Repeatedly extracts the sprite with the smallest `scale` (farthest away) and appends it to the `vsprsortedhead` list. This is an O(n^2) selection sort - acceptable because `MAXVISSPRITES` is only 128.
3. The resulting `vsprsortedhead` list is ordered from farthest to nearest; drawing front-to-back with this list gives correct overdraw (sprites farther away drawn first, closer sprites on top).

---

### `R_DrawSprite`

```c
void R_DrawSprite(vissprite_t* spr)
```

**Purpose:** Draws a single world sprite, clipping it against the drawseg (wall) records to correctly occlude sprites behind walls.

**Parameters:**
- `spr` - The vissprite to draw.

**Return value:** None.

**Key logic:**
1. Initializes local `clipbot[x]` and `cliptop[x]` arrays to the sentinel -2 for all columns in `[spr->x1, spr->x2]`.
2. Scans the `drawsegs[]` array in reverse (end to start) for drawsegs that overlap the sprite's x-range and have a silhouette or masked texture.
3. For each overlapping drawseg:
   - Computes the scale range `[lowscale, scale]` for the drawseg.
   - If the drawseg is entirely in front of the sprite (both `scale1` and `scale2` less than `spr->scale`): the drawseg is behind the sprite. If it has a masked mid-texture, render it now (masked textures appear between sprites in depth), then skip.
   - If the drawseg is in front, applies its silhouette clip (bottom, top, or both) to `clipbot`/`cliptop`, but only for columns not yet set (first clipper wins, since we iterate back-to-front).
4. After scanning all drawsegs, fills any unset column clips: `clipbot[x] = viewheight`, `cliptop[x] = -1`.
5. Sets `mfloorclip = clipbot`, `mceilingclip = cliptop`, then calls `R_DrawVisSprite`.

**Silhouette constants (from `r_defs.h`):**
- `SIL_NONE = 0` - no clipping
- `SIL_BOTTOM = 1` - clip sprite's bottom
- `SIL_TOP = 2` - clip sprite's top
- `SIL_BOTH = 3` - clip both top and bottom

---

### `R_DrawMasked`

```c
void R_DrawMasked(void)
```

**Purpose:** Top-level entry point for the masked (sprite + masked texture) rendering pass. Called after the solid geometry pass has completed.

**Parameters:** None.

**Return value:** None.

**Key logic:**
1. `R_SortVisSprites()` - sorts the collected vissprites.
2. Iterates `vsprsortedhead` list (farthest to nearest) and calls `R_DrawSprite` for each.
3. Iterates all `drawsegs[]` and calls `R_RenderMaskedSegRange` for any remaining masked mid-textures (two-sided linedefs with transparent middle textures, e.g. fences, grates).
4. Calls `R_DrawPlayerSprites()` last (on top of everything), but only when `viewangleoffset == 0` (i.e., not a side-camera view, which was a feature that was never fully exposed).

---

## Data Structures

### `maskdraw_t` (local, unused in final code)

```c
typedef struct {
    int x1;
    int x2;
    int column;
    int topclip;
    int bottomclip;
} maskdraw_t;
```

Defined locally in `r_things.c` but never actually used in the shipping code. Appears to be a remnant of an earlier clipping approach that was superseded by the per-column `clipbot`/`cliptop` arrays in `R_DrawSprite`.

For the actively used structures (`vissprite_t`, `spriteframe_t`, `spritedef_t`), see `r_defs.h`.

---

## Dependencies

| File / Module | Reason |
|---------------|--------|
| `doomdef.h`   | `SCREENWIDTH`, `boolean`, `byte`, `fixed_t` type definitions; `FF_FRAMEMASK`, `FF_FULLBRIGHT` frame flag masks |
| `m_swap.h`    | `SHORT()`, `LONG()` byte-swapping macros for reading patch header fields |
| `i_system.h`  | `I_Error()` for fatal runtime errors |
| `z_zone.h`    | `Z_Malloc()` for heap allocation of `sprites[]` and `spriteframes[]` arrays |
| `w_wad.h`     | `W_CacheLumpNum()` to load sprite patches; `lumpinfo[]` array for WAD lump name scanning |
| `r_local.h`   | Aggregator header pulling in all renderer internals (`r_main.h`, `r_bsp.h`, `r_segs.h`, `r_plane.h`, `r_data.h`, `r_things.h`, `r_draw.h`); provides `drawseg_t`, `drawsegs[]`, `ds_p`, column-rendering globals, projection constants |
| `doomstat.h`  | `modifiedgame` flag (PWAD loaded), `validcount`, `viewplayer`, `extralight`, `colormaps`, `fixedcolormap`, `translationtables` |
| `r_segs.c`    | `R_RenderMaskedSegRange()` called from `R_DrawSprite` and `R_DrawMasked` |
