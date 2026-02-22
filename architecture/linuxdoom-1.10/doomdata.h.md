# File Overview

`doomdata.h` defines all the on-disk data structures that DOOM loads directly from WAD file lumps. These are the raw binary formats used in the map data stored in the WAD — the "external data" in the original comment. Every structure here corresponds exactly to the binary layout of a WAD lump (packed, no padding-aware alignment tricks needed for the platforms DOOM targeted), making them suitable for direct memory-mapped reads from WAD data.

Understanding this file is essential to understanding how DOOM's levels are built: a WAD map consists of a sequence of lumps in a fixed order (defined by the `ML_*` enum), each containing an array of the corresponding structure type.

## Global Variables

This header defines no global variables.

## Functions

This header declares no functions.

## Data Structures

### Map Lump Order Enum (anonymous)

Defines the index of each lump in a map's WAD entry sequence. Every DOOM map consists of these lumps in this exact order, immediately following the map label lump:

```c
enum {
    ML_LABEL,      // 0: The map name lump (e.g., "E1M1" or "MAP01"). Separator only.
    ML_THINGS,     // 1: Entity/thing placement data (mapthing_t[]).
    ML_LINEDEFS,   // 2: Line definitions connecting vertices (maplinedef_t[]).
    ML_SIDEDEFS,   // 3: Side (wall face) definitions for linedefs (mapsidedef_t[]).
    ML_VERTEXES,   // 4: Vertex coordinates (mapvertex_t[]).
    ML_SEGS,       // 5: Line segments created by BSP splitting (mapseg_t[]).
    ML_SSECTORS,   // 6: Subsectors (BSP leaf nodes, list of segs) (mapsubsector_t[]).
    ML_NODES,      // 7: BSP tree nodes (mapnode_t[]).
    ML_SECTORS,    // 8: Sector definitions (mapsector_t[]).
    ML_REJECT,     // 9: Sector-to-sector visibility lookup table (raw bytes).
    ML_BLOCKMAP    // 10: Motion clipping grid for collision detection (raw shorts).
};
```

---

### `mapvertex_t` (struct)

A single 2D vertex in the map. The fundamental geometric primitive.

```c
typedef struct {
    short x;  // X coordinate in map units.
    short y;  // Y coordinate in map units.
} mapvertex_t;
```

---

### `mapsidedef_t` (struct)

Defines one face of a linedef — the visual appearance of a wall segment from one side.

```c
typedef struct {
    short textureoffset;      // Horizontal offset for wall textures.
    short rowoffset;          // Vertical offset for wall textures.
    char  toptexture[8];      // Upper texture name (null-padded, max 8 chars).
    char  bottomtexture[8];   // Lower texture name.
    char  midtexture[8];      // Middle texture name.
    short sector;             // Index into the sectors array for the front sector.
} mapsidedef_t;
```

---

### `maplinedef_t` (struct)

Defines a line segment between two vertices, with properties controlling its behavior.

```c
typedef struct {
    short v1;          // Index of the first vertex.
    short v2;          // Index of the second vertex.
    short flags;       // Attribute flags (see ML_* flag defines below).
    short special;     // Special action type (e.g., door triggers, conveyor belts).
    short tag;         // Sector tag this linedef activates.
    short sidenum[2];  // Index into sidedefs: [0]=front, [1]=back (-1 = one-sided).
} maplinedef_t;
```

**LineDef attribute flags:**

| Constant | Value | Meaning |
|----------|-------|---------|
| `ML_BLOCKING` | `1` | Solid obstacle; blocks all movement. |
| `ML_BLOCKMONSTERS` | `2` | Blocks monsters only (players can pass). |
| `ML_TWOSIDED` | `4` | Has a back sidedef; both sides are renderable. |
| `ML_DONTPEGTOP` | `8` | Upper texture is anchored to the top (unpegged from height changes). |
| `ML_DONTPEGBOTTOM` | `16` | Lower texture is anchored to the bottom. |
| `ML_SECRET` | `32` | Drawn as one-sided on the automap to hide its nature (secret). |
| `ML_SOUNDBLOCK` | `64` | Blocks sound propagation across this line. |
| `ML_DONTDRAW` | `128` | Never drawn on the automap. |
| `ML_MAPPED` | `256` | Already seen/revealed on the automap. |

---

### `mapsector_t` (struct)

Defines a sector — a convex region of the map with a flat floor and ceiling.

```c
typedef struct {
    short floorheight;      // Floor height in map units.
    short ceilingheight;    // Ceiling height in map units.
    char  floorpic[8];      // Flat texture name for the floor.
    char  ceilingpic[8];    // Flat texture name for the ceiling.
    short lightlevel;       // Ambient light level (0-255).
    short special;          // Special effect type (e.g., damage floor, blinking light).
    short tag;              // Tag number (matched against linedef tags for triggers).
} mapsector_t;
```

---

### `mapsubsector_t` (struct)

A BSP leaf node representing a convex subsector (a group of segs that bound a convex region within a single sector).

```c
typedef struct {
    short numsegs;   // Number of segs in this subsector.
    short firstseg;  // Index of the first seg in the segs array.
} mapsubsector_t;
```

---

### `mapseg_t` (struct)

A line segment produced by the BSP builder splitting linedefs. Segs are the fundamental rendering primitives.

```c
typedef struct {
    short v1;       // Start vertex index.
    short v2;       // End vertex index.
    short angle;    // Angle of the seg (BAM, binary angle measurement).
    short linedef;  // Parent linedef index.
    short side;     // 0 = front side, 1 = back side of the linedef.
    short offset;   // Horizontal offset from the linedef's start to this seg's start.
} mapseg_t;
```

---

### `mapnode_t` (struct)

A BSP tree interior node. The tree is traversed during rendering to determine front-to-back draw order.

```c
typedef struct {
    short x, y;             // Origin of the partition line.
    short dx, dy;           // Direction vector of the partition line.
    short bbox[2][4];       // Bounding boxes for each child: [0]=right, [1]=left.
                            // Each bbox is [BOXTOP, BOXBOTTOM, BOXLEFT, BOXRIGHT].
    unsigned short children[2]; // Child indices: [0]=right, [1]=left.
                                // If NF_SUBSECTOR bit is set, it is a subsector index.
} mapnode_t;
```

**`NF_SUBSECTOR`** (`0x8000`): When the high bit of a `children[]` entry is set, the lower 15 bits index into the subsectors array (a leaf node). Otherwise it indexes another `mapnode_t` (an interior node).

---

### `mapthing_t` (struct)

Defines a single "thing" in the map: the placement of a player start, monster, item, or decoration.

```c
typedef struct {
    short x;        // X coordinate of the thing.
    short y;        // Y coordinate of the thing.
    short angle;    // Facing angle in degrees (0=East, 90=North, 180=West, 270=South).
    short type;     // Thing type number (maps to mobjtype_t via mobjinfo[]).
    short options;  // Flags: MTF_EASY, MTF_NORMAL, MTF_HARD, MTF_AMBUSH (from doomdef.h).
} mapthing_t;
```

## Dependencies

| File | Reason |
|------|--------|
| `doomtype.h` | Provides `byte`, `boolean`, and numeric limit macros. |
| `doomdef.h` | Provides `MTF_EASY`, `MTF_NORMAL`, `MTF_HARD`, `MTF_AMBUSH` thing flags, and other global constants. |
