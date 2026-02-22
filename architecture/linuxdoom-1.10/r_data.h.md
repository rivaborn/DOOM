# File Overview

`r_data.h` is the public interface header for the rendering data management module (`r_data.c`). It declares the functions that other engine subsystems use to initialize rendering assets, retrieve texture column data, look up texture and flat numbers by name, and precache level graphics.

This header is included by virtually every renderer module (via `r_local.h`), the play module (via `p_local.h` for texture lookups during map setup), and the game module (for level precaching).

## Global Variables

This header declares no global variables. All rendering data globals (texture arrays, colormap pointer, sprite tables, etc.) are declared in `r_state.h`, which `r_data.h` includes.

## Functions

### `R_GetColumn`

```c
byte* R_GetColumn(int tex, int col);
```

**Purpose:** Returns a pointer to the raw pixel column data for column `col` of texture `tex`. This is the primary hot-path accessor called from `r_segs.c` during wall rendering and from `r_plane.c` for sky rendering.

**Parameters:**
- `tex` (`int`): Texture number (0-based index into `textures[]`).
- `col` (`int`): Column number, automatically wrapped via `texturewidthmask[tex]` to handle repeating textures.

**Return Value:** `byte*` — Pointer to pixel data. Valid until the zone cache is compacted.

---

### `R_InitData`

```c
void R_InitData(void);
```

**Purpose:** Top-level initialization for all rendering data. Loads and processes TEXTURE1/TEXTURE2 definitions, flat lumps, sprite lump headers, and the COLORMAP. Called from `R_Init` during engine startup, after `W_Init`.

---

### `R_PrecacheLevel`

```c
void R_PrecacheLevel(void);
```

**Purpose:** Preloads all graphics assets referenced by the current map level into the WAD cache, reducing in-game hitching. Skipped during demo playback.

---

### `R_FlatNumForName`

```c
int R_FlatNumForName(char* name);
```

**Purpose:** Converts a flat name (8-character WAD lump name such as "FLOOR5_1") to its runtime flat index (relative to `firstflat`). Calls `I_Error` if the flat is not found.

**Parameters:**
- `name` (`char*`): Flat name.

**Return Value:** `int` — Flat number for use in sector `floorpic`/`ceilingpic` fields and `visplane_t.picnum`.

---

### `R_TextureNumForName`

```c
int R_TextureNumForName(char* name);
```

**Purpose:** Converts a texture name to its runtime texture number. Calls `I_Error` if not found. Used during map loading (`P_LoadSideDefs`) and for switch/special texture lookups.

**Parameters:**
- `name` (`char*`): Texture name.

**Return Value:** `int` — Texture number.

---

### `R_CheckTextureNumForName`

```c
int R_CheckTextureNumForName(char* name);
```

**Purpose:** Like `R_TextureNumForName` but returns `-1` instead of erroring when the texture is not found. Returns `0` immediately for the "no texture" marker `"-"`. Used in contexts where a missing texture is acceptable (e.g., checking switch alternate textures).

**Parameters:**
- `name` (`char*`): Texture name.

**Return Value:** `int` — Texture number, `0` for null texture, `-1` if not found.

## Data Structures

No data structures are defined in this header. The core rendering data structures (`texture_t`, `texpatch_t`, `mappatch_t`) are defined locally in `r_data.c` as they are not needed by external code.

## Dependencies

| File | Reason |
|---|---|
| `r_defs.h` | Provides the data structure definitions (`sector_t`, `side_t`, `drawseg_t`, `vissprite_t`, etc.) that the rendering subsystem relies on |
| `r_state.h` | Declares the external rendering state variables (`textureheight`, `spritewidth`, `colormaps`, `firstflat`, `flattranslation`, `texturetranslation`, etc.) that are defined in `r_data.c` but shared across the renderer |
