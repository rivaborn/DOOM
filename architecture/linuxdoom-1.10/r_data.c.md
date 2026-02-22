# File Overview

`r_data.c` is the rendering data management module. It handles all loading, caching, and lookup of the graphical assets used during rendering: wall textures, floor/ceiling flats, sprites, and the lighting colormap. This module is responsible for bridging the WAD file format (raw binary data) and the runtime data structures the renderer uses each frame.

Key responsibilities:

- **Texture composition**: DOOM wall textures are not stored as monolithic images; instead they are defined in `TEXTURE1`/`TEXTURE2` lumps as lists of patches (sub-images) with offsets. `r_data.c` reads these definitions and builds the composite lookup tables that allow `r_segs.c` to fetch individual texture columns efficiently.
- **Flat initialization**: Flats (floor/ceiling textures) are 64x64 raw pixel arrays stored as individual lumps between `F_START` and `F_END` markers.
- **Sprite lump initialization**: Reads sprite lump metadata (width, offset, top-offset) so sprites can be rendered without loading every lump into cache during initialization.
- **Colormap loading**: Loads the `COLORMAP` lump, which provides 32 pre-computed 256-entry palette remapping tables for distance-based lighting (the further a pixel, the darker it appears).
- **Precaching**: Optionally loads all graphics that will be needed for the current level into the cache to avoid hitching during play.
- **Texture name lookup**: Provides `R_TextureNumForName` and `R_FlatNumForName` for mapping asset names to runtime indices.

## Global Variables

### Flat bookkeeping

| Type | Name | Description |
|---|---|---|
| `int` | `firstflat` | WAD lump index of the first flat (one after the `F_START` marker lump). |
| `int` | `lastflat` | WAD lump index of the last flat (one before `F_END`). |
| `int` | `numflats` | Total number of flats (`lastflat - firstflat + 1`). |

### Patch bookkeeping

| Type | Name | Description |
|---|---|---|
| `int` | `firstpatch` | WAD lump index of the first patch. |
| `int` | `lastpatch` | WAD lump index of the last patch. |
| `int` | `numpatches` | Total patch count. |

### Sprite lump bookkeeping

| Type | Name | Description |
|---|---|---|
| `int` | `firstspritelump` | WAD lump index of the first sprite (one after `S_START`). |
| `int` | `lastspritelump` | WAD lump index of the last sprite (one before `S_END`). |
| `int` | `numspritelumps` | Total sprite lump count. |

### Texture tables

| Type | Name | Description |
|---|---|---|
| `int` | `numtextures` | Total number of composite wall textures (from TEXTURE1 + TEXTURE2). |
| `texture_t**` | `textures` | Array of pointers to `texture_t` structs, indexed by texture number. Each `texture_t` describes width, height, and the list of patches that compose it. |
| `int*` | `texturewidthmask` | Per-texture bitmask for wrapping column indices (texture width is always a power of two; mask = width - 1). |
| `fixed_t*` | `textureheight` | Per-texture height in fixed-point units (used for texture pegging calculations). |
| `int*` | `texturecompositesize` | Per-texture total byte size of the composite block (only for textures with multi-patch columns). |
| `short**` | `texturecolumnlump` | For each texture, an array indexed by column number giving the WAD lump that contains that column's pixel data. `-1` if the column is a composite of multiple patches. |
| `unsigned short**` | `texturecolumnofs` | For each texture, an array indexed by column giving the byte offset within `texturecolumnlump[tex][col]` (or within the composite block if lump == -1). |
| `byte**` | `texturecomposite` | Per-texture pointer to the generated composite column cache block (in zone memory). `NULL` until first needed. |

### Translation tables for animation

| Type | Name | Description |
|---|---|---|
| `int*` | `flattranslation` | Maps flat index to the actual flat index to use, allowing animated flats to redirect all references to the current animation frame. |
| `int*` | `texturetranslation` | Maps texture index to the actual texture to use; supports animated textures and switch-state changes. |

### Sprite dimension tables

| Type | Name | Description |
|---|---|---|
| `fixed_t*` | `spritewidth` | Per-sprite-lump width in fixed-point units; avoids loading the full lump just to get the width. |
| `fixed_t*` | `spriteoffset` | Per-sprite-lump left offset (horizontal origin) in fixed-point. |
| `fixed_t*` | `spritetopoffset` | Per-sprite-lump top offset (vertical origin) in fixed-point. |

### Lighting

| Type | Name | Description |
|---|---|---|
| `lighttable_t*` | `colormaps` | Pointer to the loaded `COLORMAP` lump (256-byte-aligned). Contains 32 consecutive 256-entry palette remapping tables, from fully lit (table 0) to near-black (table 31). |

### Precache statistics

| Type | Name | Description |
|---|---|---|
| `int` | `flatmemory` | Bytes of flat data loaded during precaching (diagnostic). |
| `int` | `texturememory` | Bytes of texture patch data loaded during precaching. |
| `int` | `spritememory` | Bytes of sprite data loaded during precaching. |

## Functions

### `R_DrawColumnInCache`

```c
void R_DrawColumnInCache(column_t* patch, byte* cache, int originy, int cacheheight)
```

**Purpose:** Copies pixel data from a single patch column into the composite texture cache block, clipping to the cache height and applying the patch's vertical origin offset.

**Parameters:**
- `patch` (`column_t*`): Pointer to the start of the patch's column data (post list, terminated by `topdelta == 0xff`).
- `cache` (`byte*`): Pointer to the destination buffer in the composite block.
- `originy` (`int`): Vertical origin offset of the patch within the texture (from `texpatch_t.originy`).
- `cacheheight` (`int`): Total height of the texture in pixels; used as a clip limit.

**Return Value:** `void`

**Key Logic:** Walks the patch's post list. For each post, computes the destination position as `originy + post->topdelta`, clips to `[0, cacheheight)`, and `memcpy`s the pixel bytes into the cache.

---

### `R_GenerateComposite`

```c
void R_GenerateComposite(int texnum)
```

**Purpose:** Builds the composite pixel data for texture `texnum` by drawing all its contributing patches into a single allocated block. Called lazily the first time a multi-patch column is needed.

**Parameters:**
- `texnum` (`int`): Texture index into `textures[]`.

**Return Value:** `void`

**Key Logic:**

1. Allocates a `PU_STATIC` zone block of `texturecompositesize[texnum]` bytes and stores the pointer in `texturecomposite[texnum]`.
2. For each patch in the texture's patch list, loads the patch with `W_CacheLumpNum(PU_CACHE)`.
3. For each column `x` in the patch's horizontal extent, if `collump[x] < 0` (meaning this column requires compositing from multiple patches), calls `R_DrawColumnInCache` to draw the patch's contribution into the composite block at `colofs[x]`.
4. After all patches are composited, tags the block as `PU_CACHE` so it can be evicted under memory pressure.

---

### `R_GenerateLookup`

```c
void R_GenerateLookup(int texnum)
```

**Purpose:** Builds the `texturecolumnlump` and `texturecolumnofs` lookup tables for texture `texnum`. For each texture column, determines whether it is covered by exactly one patch (in which case the renderer can use the patch lump directly) or multiple patches (requiring the composite cache).

**Parameters:**
- `texnum` (`int`): Texture index.

**Return Value:** `void`

**Key Logic:**

1. Uses a stack-allocated `patchcount[]` array (one byte per column) to count how many patches contribute to each column.
2. For single-patch columns: sets `collump[x] = patch->patch` and `colofs[x] = offset_into_lump + 3`.
3. For multi-patch columns: sets `collump[x] = -1` and assigns a sequential offset in the composite block (`colofs[x] = texturecompositesize[texnum]`), then increments the size by `texture->height`. Errors if the composite would exceed 64K.

---

### `R_GetColumn`

```c
byte* R_GetColumn(int tex, int col)
```

**Purpose:** The primary texture column accessor used by the renderer. Returns a pointer to the pixel data for column `col` of texture `tex`.

**Parameters:**
- `tex` (`int`): Texture number.
- `col` (`int`): Column number (wrapped via `texturewidthmask[tex]` to handle texture repetition).

**Return Value:** `byte*` — Pointer directly into the WAD cache or composite block. Valid until the cache is evicted.

**Key Logic:**

1. `col &= texturewidthmask[tex]` — wraps column index for repeating textures.
2. `lump = texturecolumnlump[tex][col]`. If `lump > 0`, the column is in a single patch lump: return `W_CacheLumpNum(lump, PU_CACHE) + ofs`.
3. If `lump <= 0`, check if the composite block is built. If not, call `R_GenerateComposite(tex)`. Return `texturecomposite[tex] + ofs`.

---

### `R_InitTextures`

```c
void R_InitTextures(void)
```

**Purpose:** Loads and parses the `TEXTURE1` (and optionally `TEXTURE2`) WAD lumps, building the `textures[]` array, all per-texture lookup tables, and the `texturetranslation` identity mapping.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:**

1. Loads the `PNAMES` lump to map patch names to WAD lump numbers.
2. Loads `TEXTURE1` (and `TEXTURE2` if present in the WAD — i.e., full commercial DOOM II).
3. For each `maptexture_t` entry, allocates a `texture_t` struct with embedded `texpatch_t` array and fills in dimensions, patch count, and patch details (resolving patch names to lump numbers via `patchlookup`).
4. For each texture, allocates `texturecolumnlump` and `texturecolumnofs` arrays (2 bytes per column), computes `texturewidthmask` and `textureheight`.
5. Calls `R_GenerateLookup` for every texture.
6. Allocates `texturetranslation` as an identity mapping.
7. Prints progress dots to stdout.

---

### `R_InitFlats`

```c
void R_InitFlats(void)
```

**Purpose:** Locates the flat lumps between `F_START` and `F_END` markers in the WAD and builds the identity `flattranslation` array.

**Parameters:** None.

**Return Value:** `void`

---

### `R_InitSpriteLumps`

```c
void R_InitSpriteLumps(void)
```

**Purpose:** Locates sprite lumps between `S_START` and `S_END` and reads the width, left-offset, and top-offset of each sprite into `spritewidth[]`, `spriteoffset[]`, and `spritetopoffset[]`. This avoids needing to load entire sprite lumps just to get their headers during rendering.

**Parameters:** None.

**Return Value:** `void`

---

### `R_InitColormaps`

```c
void R_InitColormaps(void)
```

**Purpose:** Loads the `COLORMAP` lump into 256-byte-aligned zone memory.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:** Allocates `lump_length + 255` bytes, then aligns the pointer to the next 256-byte boundary using `((int)ptr + 255) & ~0xff`. This alignment is required because the colormap is indexed with `lightlevel * 256` arithmetic.

---

### `R_InitData`

```c
void R_InitData(void)
```

**Purpose:** Top-level initialization function that calls all the sub-initialization routines in order. Called from `R_Init` during startup.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:** Calls `R_InitTextures`, `R_InitFlats`, `R_InitSpriteLumps`, `R_InitColormaps` in sequence, printing progress messages between each.

---

### `R_FlatNumForName`

```c
int R_FlatNumForName(char* name)
```

**Purpose:** Converts a flat name (up to 8 characters) to a flat index (relative to `firstflat`).

**Parameters:**
- `name` (`char*`): The flat name.

**Return Value:** `int` — Flat index, or calls `I_Error` if not found.

---

### `R_CheckTextureNumForName`

```c
int R_CheckTextureNumForName(char* name)
```

**Purpose:** Looks up a texture by name. Returns `-1` if not found instead of erroring. Returns `0` immediately for the "no texture" marker (`name[0] == '-'`).

**Parameters:**
- `name` (`char*`): Texture name, up to 8 characters.

**Return Value:** `int` — Texture number, `0` for null-texture, `-1` if not found.

**Key Logic:** Linear search through `textures[0..numtextures-1]` using `strncasecmp`.

---

### `R_TextureNumForName`

```c
int R_TextureNumForName(char* name)
```

**Purpose:** Like `R_CheckTextureNumForName` but calls `I_Error` if the texture is not found.

**Parameters:**
- `name` (`char*`): Texture name.

**Return Value:** `int` — Texture number.

---

### `R_PrecacheLevel`

```c
void R_PrecacheLevel(void)
```

**Purpose:** Preloads all textures, flats, and sprites that are referenced by the current map into the WAD cache. Reduces or eliminates hitching during play caused by on-demand lump loading.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:**

1. Returns immediately if `demoplayback` (no need to precache for demos).
2. **Flats:** Scans all sectors for used floor/ceiling flats; calls `W_CacheLumpNum(PU_CACHE)` on each.
3. **Textures:** Scans all sidedefs for used top/mid/bottom textures plus `skytexture`; for each texture loads all its component patches.
4. **Sprites:** Scans the thinker list for all live `mobj_t` instances; for each unique sprite type, loads all 8 rotation frames of all animation frames.
5. Accumulates byte counts into `flatmemory`, `texturememory`, `spritememory` for diagnostic output.

## Data Structures

### `mappatch_t` (local)

```c
typedef struct {
    short originx;
    short originy;
    short patch;
    short stepdir;
    short colormap;
} mappatch_t;
```

Raw WAD format for one patch entry within a `maptexture_t`. `patch` is an index into the `PNAMES` list. `stepdir` and `colormap` are fields present in the format but unused by DOOM (holdovers from earlier design).

### `maptexture_t` (local)

```c
typedef struct {
    char        name[8];
    boolean     masked;
    short       width;
    short       height;
    void**      columndirectory;  // OBSOLETE
    short       patchcount;
    mappatch_t  patches[1];
} maptexture_t;
```

Raw WAD format texture definition. `columndirectory` is commented as obsolete. The `patches` array is a flexible array of `patchcount` entries.

### `texpatch_t` (local)

```c
typedef struct {
    int originx;
    int originy;
    int patch;
} texpatch_t;
```

Runtime representation of a single patch within a texture. Unlike `mappatch_t`, the fields are widened to `int` and unnecessary fields are dropped.

### `texture_t` (local)

```c
typedef struct {
    char        name[8];
    short       width;
    short       height;
    short       patchcount;
    texpatch_t  patches[1];
} texture_t;
```

Runtime texture definition. An array of `patchcount` `texpatch_t` entries follows the struct in memory.

## Dependencies

| File | Reason |
|---|---|
| `i_system.h` | `I_Error()` for fatal errors |
| `z_zone.h` | `Z_Malloc`, `Z_Free`, `Z_ChangeTag` for zone memory management |
| `m_swap.h` | `SHORT()`, `LONG()` macros for endian-swapping WAD data |
| `w_wad.h` | `W_CacheLumpName`, `W_CacheLumpNum`, `W_GetNumForName`, `W_CheckNumForName`, `W_LumpLength`, `W_ReadLump`, `lumpinfo[]` |
| `doomdef.h` | Core type definitions |
| `r_local.h` | Full renderer header aggregation (includes `r_main.h`, `r_defs.h`, etc.) |
| `p_local.h` | `P_MobjThinker` for sprite precaching; `thinkercap` thinker list access |
| `doomstat.h` | `demoplayback`, `numsectors`, `numsides`, `numsprites`, `sprites`, `modifiedgame` |
| `r_sky.h` | `skytexture` for precaching sky texture |
| `r_data.h` | Self-reference (function declarations) |
| `<alloca.h>` | Stack allocation for temporary arrays (Linux-specific) |
