# File Overview

`m_bbox.c` implements DOOM's axis-aligned bounding box (AABB) utility functions. Bounding boxes are used extensively throughout the renderer and physics system to perform fast spatial queries — determining whether two objects could possibly intersect, whether a line segment crosses a region, whether a point is inside an area, etc.

A bounding box in DOOM is represented as a fixed-size array of four `fixed_t` values indexed by the `BOXTOP`, `BOXBOTTOM`, `BOXLEFT`, and `BOXRIGHT` constants defined in `m_bbox.h`. This compact representation allows bounding boxes to be stored inline in map structures and passed by pointer efficiently.

The module provides only two operations:
1. **Clear** — reset a box to represent an empty (infinite-minimum) region
2. **Expand** — extend a box to include a given point

These primitives are composed to build the bounding boxes of complex objects (like BSP nodes, linedefs, and map sectors) by iterating over the object's vertices.

## Global Variables

No global variables are defined in this file.

## Functions

### `M_ClearBox`

**Signature:** `void M_ClearBox(fixed_t *box)`

**Purpose:** Resets a bounding box to an "empty" state, such that any subsequent call to `M_AddToBox` will correctly establish the box around the first point added.

**Parameters:**
- `box` - Pointer to a 4-element `fixed_t` array in `[BOXTOP, BOXBOTTOM, BOXLEFT, BOXRIGHT]` order

**Return value:** None.

**Key logic:**
- Sets `box[BOXTOP]` and `box[BOXRIGHT]` to `MININT` (the smallest possible integer value). Because the top and right bounds start at the minimum, any real point will be larger and will expand the box.
- Sets `box[BOXBOTTOM]` and `box[BOXLEFT]` to `MAXINT` (the largest possible integer value). Because the bottom and left bounds start at the maximum, any real point will be smaller and will expand the box.

This is the standard approach for initializing a bounding box before accumulating points: the box begins as logically empty (inverted bounds), and the first `M_AddToBox` call establishes it.

---

### `M_AddToBox`

**Signature:** `void M_AddToBox(fixed_t* box, fixed_t x, fixed_t y)`

**Purpose:** Expands the bounding box to include the point `(x, y)`. If the point is already inside the box, the box is unchanged. If it is outside, the nearest wall(s) of the box are pushed outward to include the point.

**Parameters:**
- `box` - Pointer to a 4-element `fixed_t` array representing the bounding box
- `x` - The X coordinate (in `fixed_t` map units) of the point to include
- `y` - The Y coordinate (in `fixed_t` map units) of the point to include

**Return value:** None.

**Key logic:**
- If `x < box[BOXLEFT]`: update `box[BOXLEFT] = x` (extend left boundary)
- Else if `x > box[BOXRIGHT]`: update `box[BOXRIGHT] = x` (extend right boundary)
- If `y < box[BOXBOTTOM]`: update `box[BOXBOTTOM] = y` (extend bottom boundary)
- Else if `y > box[BOXTOP]`: update `box[BOXTOP] = y` (extend top boundary)

Note: DOOM uses a standard mathematical coordinate system in which Y increases upward, so `BOXTOP` has the larger Y value and `BOXBOTTOM` has the smaller Y value.

## Data Structures

No new data structures are defined in this file. The bounding box itself is simply a `fixed_t[4]` array. The index constants (`BOXTOP`, `BOXBOTTOM`, `BOXLEFT`, `BOXRIGHT`) are defined in `m_bbox.h` as an anonymous enum.

## Dependencies

| File | Reason |
|------|--------|
| `m_bbox.h` | Own header: provides `fixed_t` type (via `m_fixed.h`), the `BOX*` index constants, and the function declarations |
| `<values.h>` | (Included via `m_bbox.h`) Provides `MININT` and `MAXINT` constants used in `M_ClearBox` |
