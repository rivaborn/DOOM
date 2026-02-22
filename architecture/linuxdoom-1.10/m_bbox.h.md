# File Overview

`m_bbox.h` is the header for DOOM's axis-aligned bounding box module. It defines the index constants for the four sides of a bounding box and declares the two bounding box manipulation functions.

Bounding boxes (AABBs) are a fundamental spatial acceleration structure used throughout the DOOM engine. They are stored as fixed-size `fixed_t[4]` arrays and indexed by the named constants defined here. The coordinate system follows DOOM's standard convention where Y increases upward (north on the map).

Bounding boxes appear in:
- BSP tree node bounds (for fast line/box intersection tests during rendering)
- Mobj (thing) spatial queries in `p_map.c` (for blockmap traversal)
- Line/sector bounds in `p_setup.c`
- Automap rendering bounds

## Global Variables

No global variables are declared in this header.

## Functions

### `M_ClearBox`

**Signature:** `void M_ClearBox(fixed_t* box)`

**Purpose:** Initializes a bounding box to an empty/infinite state. After clearing, any point added with `M_AddToBox` will correctly establish the initial bounds.

**Parameters:**
- `box` - Pointer to a 4-element `fixed_t` array

**Return value:** None.

---

### `M_AddToBox`

**Signature:** `void M_AddToBox(fixed_t* box, fixed_t x, fixed_t y)`

**Purpose:** Expands the bounding box to include the specified point. If the point is already inside the box, no change is made.

**Parameters:**
- `box` - Pointer to the bounding box to expand
- `x` - X coordinate to include (in map fixed-point units)
- `y` - Y coordinate to include (in map fixed-point units)

**Return value:** None.

## Data Structures

### Bounding Box Index Enum

```c
enum
{
    BOXTOP,       // Index 0: maximum Y coordinate (northern edge)
    BOXBOTTOM,    // Index 1: minimum Y coordinate (southern edge)
    BOXLEFT,      // Index 2: minimum X coordinate (western edge)
    BOXRIGHT      // Index 3: maximum X coordinate (eastern edge)
};
```

This anonymous enum defines the array indices used to access the four edges of a bounding box stored as `fixed_t[4]`. The ordering (top, bottom, left, right = 0, 1, 2, 3) is fixed by convention and must be preserved across all uses.

A typical bounding box usage pattern:
```c
fixed_t bbox[4];
M_ClearBox(bbox);
M_AddToBox(bbox, vertex1.x, vertex1.y);
M_AddToBox(bbox, vertex2.x, vertex2.y);
// bbox[BOXTOP] is now the maximum Y, bbox[BOXRIGHT] is the maximum X, etc.
```

## Dependencies

| File | Reason |
|------|--------|
| `<values.h>` | Provides `MININT` and `MAXINT` constants used by `M_ClearBox` in the implementation |
| `m_fixed.h` | Provides the `fixed_t` type definition (16.16 fixed-point integer) used as the bounding box coordinate type |
