# File Overview

`r_defs.h` is the master data structure definition header shared between the rendering (refresh) subsystem and the play (game logic) subsystem. It defines all the runtime map types and rendering auxiliary types that both subsystems need to access simultaneously.

This is one of the most important headers in the codebase. Nearly every other module includes it (directly or transitively via `r_local.h`, `r_data.h`, or `p_local.h`) because it defines:

- The fundamental map geometry types: `vertex_t`, `sector_t`, `side_t`, `line_t`, `subsector_t`, `seg_t`, `node_t`
- Supporting renderer types: `drawseg_t`, `patch_t`, `vissprite_t`, `visplane_t`
- Sprite definition types: `spriteframe_t`, `spritedef_t`
- Raw pixel data types: `post_t`, `column_t`, `lighttable_t`

The co-location of map geometry types and rendering types in a single header reflects DOOM's architecture where the renderer and the game engine share the same world data directly, without an intermediate scene-graph abstraction.

## Global Variables

This header defines no global variables (only type definitions and constants).

## Data Structures

### Silhouette constants

```c
#define SIL_NONE    0
#define SIL_BOTTOM  1
#define SIL_TOP     2
#define SIL_BOTH    3
```

Used in `drawseg_t.silhouette` to indicate which edges of a wall segment can clip sprites. `SIL_BOTTOM` means the wall's bottom edge is a silhouette boundary; `SIL_TOP` means the top edge is; `SIL_BOTH` is used for solid single-sided walls.

```c
#define MAXDRAWSEGS  256
```

Maximum number of visible wall segments per frame. Limits the `drawsegs[]` array.

---

### `vertex_t`

```c
typedef struct {
    fixed_t x;
    fixed_t y;
} vertex_t;
```

A 2D map vertex. Coordinates are in fixed-point world units. Stored in the `VERTEXES` WAD lump. Referenced by `seg_t` and `line_t`.

---

### `degenmobj_t`

```c
typedef struct {
    thinker_t   thinker;  // not used for anything
    fixed_t     x;
    fixed_t     y;
    fixed_t     z;
} degenmobj_t;
```

A degenerate map object used as a sound origin for sectors. Sectors embed one of these in their `soundorg` field so that the sound system can treat a sector as a sound source using the same interface as real `mobj_t` objects. The `thinker` field is unused.

---

### `sector_t`

```c
typedef struct {
    fixed_t     floorheight;
    fixed_t     ceilingheight;
    short       floorpic;        // flat index
    short       ceilingpic;      // flat index
    short       lightlevel;      // 0-255
    short       special;         // sector special type
    short       tag;             // for line->sector matching
    int         soundtraversed;  // 0=untraversed, 1,2=sndlines-1
    mobj_t*     soundtarget;     // thing that made a sound
    int         blockbox[4];     // blockmap bounding box
    degenmobj_t soundorg;        // sound origin in center
    int         validcount;      // if==validcount, already checked
    mobj_t*     thinglist;       // list of mobjs in sector
    void*       specialdata;     // thinker for reversible actions
    int         linecount;
    struct line_s** lines;       // [linecount] array of lines
} sector_t;
```

The runtime sector record. A sector is a convex polygon region of the map with a flat floor, flat ceiling, uniform light level, and optional special behavior. The `thinglist` is a linked list of all `mobj_t` currently in the sector. `specialdata` points to an active thinker (e.g., a ceiling or floor mover) if one is attached.

---

### `side_t`

```c
typedef struct {
    fixed_t     textureoffset;   // column offset added to texture mapping
    fixed_t     rowoffset;       // row offset added to texture top
    short       toptexture;      // texture index for upper wall
    short       bottomtexture;   // texture index for lower wall
    short       midtexture;      // texture index for middle wall
    sector_t*   sector;          // sector this sidedef faces
} side_t;
```

A sidedef defines the visual appearance of one face of a linedef. Two-sided linedefs have two sidedefs. `textureoffset` and `rowoffset` implement scrollable textures and allow fine-tuning of texture alignment. Texture indices of `0` indicate no texture (sky or invisible).

---

### `slopetype_t`

```c
typedef enum {
    ST_HORIZONTAL,
    ST_VERTICAL,
    ST_POSITIVE,
    ST_NEGATIVE
} slopetype_t;
```

Classification of a linedef's slope for fast collision detection. Axis-aligned lines use faster integer comparisons; diagonal lines use the appropriate sign convention.

---

### `line_t` (struct `line_s`)

```c
typedef struct line_s {
    vertex_t*   v1;
    vertex_t*   v2;
    fixed_t     dx;         // v2->x - v1->x (precomputed)
    fixed_t     dy;         // v2->y - v1->y (precomputed)
    short       flags;      // ML_* flags (blocking, secret, mapped, etc.)
    short       special;    // line special type
    short       tag;        // tag for sector matching
    short       sidenum[2]; // side 0 = front, side 1 = back (-1 if one-sided)
    fixed_t     bbox[4];    // bounding box
    slopetype_t slopetype;
    sector_t*   frontsector;
    sector_t*   backsector; // NULL for one-sided lines
    int         validcount; // anti-repeat check for traversals
    void*       specialdata;// active thinker for switches etc.
} line_t;
```

A linedef connects two vertices. One-sided linedefs divide open map space from void; two-sided linedefs divide sectors from each other. `flags` controls properties like impassability, secret status, and texture unpegging. `special` and `tag` implement interactive behaviors (doors, crushers, teleporters, etc.).

---

### `subsector_t`

```c
typedef struct subsector_s {
    sector_t*   sector;
    short       numlines;
    short       firstline;  // index into segs[]
} subsector_t;
```

A BSP leaf. Represents a convex polygon (guaranteed by BSP construction) within a sector. Contains a list of `numlines` contiguous segments starting at `segs[firstline]`. The renderer processes one subsector at a time in front-to-back BSP order.

---

### `seg_t`

```c
typedef struct {
    vertex_t*   v1;
    vertex_t*   v2;
    fixed_t     offset;       // horizontal texture offset along wall
    angle_t     angle;        // angle from v1 to v2
    side_t*     sidedef;
    line_t*     linedef;
    sector_t*   frontsector;
    sector_t*   backsector;   // NULL for one-sided
} seg_t;
```

A line segment from the `SEGS` WAD lump. Segs are the basic rendering primitive — each is a portion of a linedef's sidedef visible from a specific subsector. `offset` accounts for the starting horizontal position within the parent linedef's texture.

---

### `node_t`

```c
typedef struct {
    fixed_t         x, y;          // partition line origin
    fixed_t         dx, dy;        // partition line direction
    fixed_t         bbox[2][4];    // bounding boxes: [0]=front, [1]=back
    unsigned short  children[2];   // child indices; high bit = subsector
} node_t;
```

An internal BSP node. The partition line divides the node's space into front (`children[0]`) and back (`children[1]`) halves. If a child index has the `NF_SUBSECTOR` bit set (0x8000), the lower bits are a subsector index rather than a node index.

---

### `post_t` and `column_t`

```c
typedef struct {
    byte topdelta;  // top of run; 0xff = last post
    byte length;    // number of pixels in this run; pixel data follows
} post_t;

typedef post_t column_t;
```

The compressed vertical column format used for all wall textures and sprites. A column is a sequence of posts (opaque runs separated by transparent gaps). `topdelta` is the starting row of the run; `length` is the number of pixels. The pixel bytes immediately follow the `length` byte in memory.

---

### `lighttable_t`

```c
typedef byte lighttable_t;
```

A single entry in a colormap. Arrays of 256 of these form one colormap level (256-byte palette remap table). The type alias provides semantic clarity.

---

### `drawseg_t`

```c
typedef struct drawseg_s {
    seg_t*      curline;
    int         x1, x2;             // screen column range
    fixed_t     scale1, scale2;     // scale at x1 and x2
    fixed_t     scalestep;          // (scale2-scale1)/(x2-x1)
    int         silhouette;         // SIL_* flags for sprite clipping
    fixed_t     bsilheight;         // bottom silhouette height
    fixed_t     tsilheight;         // top silhouette height
    short*      sprtopclip;         // upper clip array for sprite drawing
    short*      sprbottomclip;      // lower clip array for sprite drawing
    short*      maskedtexturecol;   // column offsets for masked midtexture
} drawseg_t;
```

A record of one rendered wall segment range, stored in `drawsegs[]`. Used during sprite rendering (`R_DrawSprite` in `r_things.c`) to determine which wall segments occlude which sprite columns. The clip arrays point into the `openings[]` buffer in `r_plane.c`.

---

### `patch_t`

```c
typedef struct {
    short   width;
    short   height;
    short   leftoffset;   // pixels to left of origin
    short   topoffset;    // pixels below origin
    int     columnofs[8]; // actually [width] entries
} patch_t;
```

Header for a WAD patch (sprite or masked texture graphic). `columnofs[i]` is a byte offset from the start of the patch to the `column_t` data for column `i`. The array is declared with 8 entries for C struct sizing purposes, but actually has `width` entries at runtime.

---

### `vissprite_t`

```c
typedef struct vissprite_s {
    struct vissprite_s* prev;
    struct vissprite_s* next;
    int         x1, x2;        // screen column range
    fixed_t     gx, gy;        // world position
    fixed_t     gz, gzt;       // world bottom/top for silhouette
    fixed_t     startfrac;     // horizontal texture fraction at x1
    fixed_t     scale;         // projection scale at center
    fixed_t     xiscale;       // horizontal step (negative if flipped)
    fixed_t     texturemid;    // vertical texture mid for column drawing
    int         patch;         // sprite lump number
    lighttable_t* colormap;    // NULL = fuzz (shadow) effect
    int         mobjflags;     // flags from the source mobj
} vissprite_t;
```

Represents a projected sprite pending drawing this frame. Stored in the `vissprites[]` array. Linked into a sorted list (front-to-back by `scale`) by `R_SortVisSprites` for correct overdraw.

---

### `spriteframe_t`

```c
typedef struct {
    boolean rotate;     // false = use lump[0] for all views
    short   lump[8];    // sprite lump offsets for rotations 0-7
    byte    flip[8];    // 1 = horizontally mirror this rotation
} spriteframe_t;
```

Describes one animation frame of a sprite, which may have up to 8 rotations. If `rotate == false`, all eight `lump[]` entries contain the same value (the single full-rotation lump). Mirrored rotations are indicated by `flip[]` set to 1.

---

### `spritedef_t`

```c
typedef struct {
    int             numframes;
    spriteframe_t*  spriteframes;  // [numframes] array
} spritedef_t;
```

Defines all animation frames for one sprite type (e.g., `SPR_TROO` for the Imp). The `sprites[]` global array contains one `spritedef_t` per sprite type enumerated in `spritenum_t`.

---

### `visplane_t`

```c
typedef struct {
    fixed_t height;
    int     picnum;
    int     lightlevel;
    int     minx, maxx;
    byte    pad1;
    byte    top[SCREENWIDTH];    // top y pixel for each column
    byte    pad2;
    byte    pad3;
    byte    bottom[SCREENWIDTH]; // bottom y pixel for each column
    byte    pad4;
} visplane_t;
```

Represents a floor or ceiling plane that spans multiple screen columns and needs horizontal span-drawing during `R_DrawPlanes`. Each visplane is uniquely identified by `(height, picnum, lightlevel)`. The `top[]` and `bottom[]` arrays record which rows need to be filled for each column. The `pad` bytes exist to prevent out-of-bounds writes to adjacent array elements during the span-building loop in `R_MakeSpans`.

## Dependencies

| File | Reason |
|---|---|
| `doomdef.h` | `SCREENWIDTH`, `byte`, `boolean`, and other core definitions |
| `m_fixed.h` | `fixed_t` type |
| `d_think.h` | `thinker_t` (embedded in `degenmobj_t`) |
| `p_mobj.h` | `mobj_t` forward declaration/definition (sectors hold `mobj_t*` links) |
