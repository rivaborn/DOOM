# File Overview

`p_maputl.c` provides the low-level geometric and spatial utility functions that underpin the DOOM collision detection and ray-casting systems. It is the foundational layer on which `p_map.c` is built.

The file's responsibilities fall into four groups:

1. **Point and box geometry**: fast approximate distance, point-on-line-side tests (for both `line_t` and `divline_t` representations), box-on-line-side classification, and line opening calculations for two-sided linedefs.

2. **Thing position management**: linking and unlinking `mobj_t` objects into the dual spatial databases - the sector thing list (used by the renderer) and the blockmap thing chain (used by collision).

3. **Blockmap iterators**: `P_BlockLinesIterator` and `P_BlockThingsIterator` enumerate the contents of a single 128x128 blockmap cell, calling a supplied callback function for each line or thing. This is the workhorse of all proximity queries.

4. **Path traversal**: `P_PathTraverse` implements a 2D DDA ray march through the blockmap, collecting all line and thing intercepts along a ray and then delivering them to a callback in distance order. This powers hitscan shooting, autoaim, line-use detection, and slide movement.

---

## Global Variables

### Line Opening State

| Type | Name | Description |
|------|------|-------------|
| `fixed_t` | `opentop` | Top of the passable window through the most recently tested two-sided line |
| `fixed_t` | `openbottom` | Bottom of the passable window |
| `fixed_t` | `openrange` | Height of the passable window (`opentop - openbottom`); zero for one-sided lines |
| `fixed_t` | `lowfloor` | Lower of the two floor heights at the line; used for drop-off detection |

### Intercept Traversal State

| Type | Name | Description |
|------|------|-------------|
| `intercept_t[MAXINTERCEPTS]` | `intercepts` | Fixed 128-entry array for accumulating line and thing intercepts |
| `intercept_t*` | `intercept_p` | Next available slot in `intercepts[]`; reset to `intercepts` at each `P_PathTraverse` call |
| `divline_t` | `trace` | The current ray being traced, set by `P_PathTraverse` and read by `PIT_AddLineIntercepts` and `PIT_AddThingIntercepts` |
| `boolean` | `earlyout` | If `true` and a solid one-sided line is hit, `PIT_AddLineIntercepts` returns `false` immediately |
| `int` | `ptflags` | The flags passed to `P_PathTraverse` (`PT_ADDLINES`, `PT_ADDTHINGS`, `PT_EARLYOUT`) |

---

## Functions

### `P_AproxDistance`

```c
fixed_t P_AproxDistance(fixed_t dx, fixed_t dy)
```

**Purpose:** Computes an inexpensive approximation of the Euclidean distance `sqrt(dx^2 + dy^2)`.

**Parameters:**
- `dx`, `dy` - Signed component differences.

**Return value:** An approximate distance in fixed-point world units.

**Key logic:** Uses the formula `max + min - min/2` (i.e., `max(|dx|,|dy|) + min(|dx|,|dy|)/2`). This is accurate to within approximately 6% of the true distance and requires no multiplication or division. Used extensively for sound attenuation, monster AI, and missile targeting where exact distance is not critical.

---

### `P_PointOnLineSide`

```c
int P_PointOnLineSide(fixed_t x, fixed_t y, line_t* line)
```

**Purpose:** Determines which side of a map line a point lies on.

**Parameters:**
- `x`, `y` - The point in world coordinates.
- `line` - The line to test against.

**Return value:** `0` for the front side, `1` for the back side.

**Key logic:**
- Handles axis-aligned special cases (horizontal and vertical lines) with simple comparisons to avoid the general cross-product.
- General case: computes the 2D cross product of `(line->dy, line->dx)` with `(x - v1->x, y - v1->y)` using fixed-point arithmetic with `FRACBITS` scaling: `left = line->dy * dx`, `right = dy * line->dx`. If `right < left`, the point is on the front side.

---

### `P_BoxOnLineSide`

```c
int P_BoxOnLineSide(fixed_t* tmbox, line_t* ld)
```

**Purpose:** Classifies an axis-aligned bounding box relative to an infinite line. Used to quickly discard lines that cannot possibly intersect a moving object's bounding box.

**Parameters:**
- `tmbox` - Four-element array: `[BOXTOP, BOXBOTTOM, BOXLEFT, BOXRIGHT]`.
- `ld` - The line to test.

**Return value:** `0` or `1` if the box is entirely on one side; `-1` if the box straddles the line.

**Key logic:** Dispatches on `ld->slopetype` (`ST_HORIZONTAL`, `ST_VERTICAL`, `ST_POSITIVE`, `ST_NEGATIVE`) to select which two diagonal corners of the box to test. For axis-aligned lines, direct comparisons suffice; for diagonal lines, `P_PointOnLineSide` is called on the two relevant corners. If both corners are on the same side, returns that side; otherwise returns -1.

---

### `P_PointOnDivlineSide`

```c
int P_PointOnDivlineSide(fixed_t x, fixed_t y, divline_t* line)
```

**Purpose:** Same as `P_PointOnLineSide` but operates on a `divline_t` rather than a `line_t`. Used in the intercept detection code.

**Parameters:**
- `x`, `y` - The test point.
- `line` - The divline.

**Return value:** `0` for front, `1` for back.

**Key logic:** Handles axis-aligned cases first. For the general case, uses a sign-bit optimization: if the XOR of the signs of `dy`, `dx`, `dx_delta`, `dy_delta` is set, the result can be determined from the sign of `(dy ^ dx_delta)` alone without a multiply. Falls back to fixed-point cross product using `>>8` prescaling to avoid overflow.

---

### `P_MakeDivline`

```c
void P_MakeDivline(line_t* li, divline_t* dl)
```

**Purpose:** Converts a `line_t` to a `divline_t` representation by copying vertex 1 and the delta.

**Key logic:** Trivial field copy: `dl->x = li->v1->x`, `dl->y = li->v1->y`, `dl->dx = li->dx`, `dl->dy = li->dy`.

---

### `P_InterceptVector`

```c
fixed_t P_InterceptVector(divline_t* v2, divline_t* v1)
```

**Purpose:** Returns the fractional parameter `t` at which divline `v2` intersects divline `v1`. This answers "at what fraction along `v2` does the intersection occur?"

**Parameters:**
- `v2` - The line whose intersection parameter is computed.
- `v1` - The second line.

**Return value:** The fraction `t` as a `fixed_t` (0 = start of `v2`, `FRACUNIT` = end). Returns 0 if lines are parallel.

**Key logic:** Implements the standard two-line intersection formula using fixed-point arithmetic. Values are right-shifted by 8 before multiplication to prevent 32-bit overflow. The denominator is `v1->dy * v2->dx - v1->dx * v2->dy`; the numerator involves the relative position of the origins. There is also an unused floating-point version in `#else` that served as a debug reference.

---

### `P_LineOpening`

```c
void P_LineOpening(line_t* linedef)
```

**Purpose:** Computes the vertical "opening" or passable window through a two-sided line - the range of Z values through which objects can pass.

**Parameters:**
- `linedef` - The two-sided line to analyze.

**Key logic:**
- One-sided lines: sets `openrange = 0` immediately.
- Two-sided lines: `opentop = min(front->ceilingheight, back->ceilingheight)`, `openbottom = max(front->floorheight, back->floorheight)`. The higher floor is the step an object must climb; the lower floor is `lowfloor` (the drop-off depth).
- `openrange = opentop - openbottom`. If negative or zero, nothing can pass through.

---

### `P_UnsetThingPosition`

```c
void P_UnsetThingPosition(mobj_t* thing)
```

**Purpose:** Removes a mobj from both its sector thing list and its blockmap cell chain. Must be called before changing `thing->x` or `thing->y`.

**Parameters:**
- `thing` - The mobj to unlink.

**Key logic:**
- Sector list: updates `thing->sprev->snext` and `thing->snext->sprev` to splice the mobj out. If it was the head of `sector->thinglist`, updates that pointer.
- Blockmap: updates `thing->bprev->bnext` and `thing->bnext->bprev`. If it was the head of a block cell (`bprev == NULL`), recomputes the block index and updates `blocklinks[blocky*bmapwidth+blockx]`.
- Skips both operations if `MF_NOSECTOR` or `MF_NOBLOCKMAP` are set.

---

### `P_SetThingPosition`

```c
void P_SetThingPosition(mobj_t* thing)
```

**Purpose:** Links a mobj into the sector thing list and blockmap chain at its current `x`, `y` position. Must be called after updating position.

**Parameters:**
- `thing` - The mobj to link in.

**Key logic:**
- Uses `R_PointInSubsector` to find the correct subsector; stores it in `thing->subsector`.
- Sector list (unless `MF_NOSECTOR`): inserts at the head of `sec->thinglist` using a doubly-linked list splice.
- Blockmap (unless `MF_NOBLOCKMAP`): computes block index, inserts at the head of `blocklinks[blocky*bmapwidth+blockx]`. Things off the edge of the map set `bnext = bprev = NULL`.

---

### `P_BlockLinesIterator`

```c
boolean P_BlockLinesIterator(int x, int y, boolean(*func)(line_t*))
```

**Purpose:** Iterates all `line_t` segments stored in blockmap cell `(x, y)`, calling `func` for each.

**Parameters:**
- `x`, `y` - Blockmap cell coordinates.
- `func` - Callback function; returns `false` to stop iteration.

**Return value:** `true` if all lines were checked; `false` if `func` returned `false`.

**Key logic:**
- Out-of-bounds cells return `true` immediately (no lines to check).
- Uses `validcount` to prevent processing the same line twice when it spans multiple cells: each line is checked only once per `validcount` value. The caller must increment `validcount` before the first call.
- The block's line list is stored in `blockmaplump` as a `-1`-terminated array of line indices.

---

### `P_BlockThingsIterator`

```c
boolean P_BlockThingsIterator(int x, int y, boolean(*func)(mobj_t*))
```

**Purpose:** Iterates all `mobj_t` objects whose origin is in blockmap cell `(x, y)`, calling `func` for each.

**Parameters:**
- `x`, `y` - Blockmap cell coordinates.
- `func` - Callback; returns `false` to stop.

**Return value:** `true` if all things were checked; `false` early exit.

**Key logic:** Walks `blocklinks[y*bmapwidth+x]` via `mobj->bnext`. No `validcount` guard is needed because each mobj is in exactly one block cell.

---

### `PIT_AddLineIntercepts`

```c
boolean PIT_AddLineIntercepts(line_t* ld)
```

**Purpose:** Block-lines callback for `P_PathTraverse`. Checks whether the current trace ray intersects `ld` and if so, records the intercept.

**Return value:** `true` to continue; `false` for early-out on solid lines.

**Key logic:**
1. Determines which side of `ld` each endpoint of the trace is on. If both are on the same side, the line is not crossed.
2. Large-trace optimization: when `|trace.dx|` or `|trace.dy|` exceed 16 world units (in fixed-point terms), uses `P_PointOnDivlineSide` (more overflow-safe); otherwise uses `P_PointOnLineSide`.
3. Converts `ld` to a `divline_t` and calls `P_InterceptVector` to get the fraction `frac`.
4. Discards intercepts with `frac < 0` (behind the trace origin).
5. Early-out: if `earlyout` and `frac < FRACUNIT` and `ld` is one-sided, returns `false`.
6. Records the intercept in `*intercept_p++`.

---

### `PIT_AddThingIntercepts`

```c
boolean PIT_AddThingIntercepts(mobj_t* thing)
```

**Purpose:** Block-things callback for `P_PathTraverse`. Checks whether the current trace ray passes through a thing's bounding "cross-line" and records the intercept.

**Key logic:**
1. Models the thing as a diagonal line segment connecting two opposing corners of its bounding square. The chosen diagonal is `positive` if `trace.dx ^ trace.dy > 0`, else `negative`.
2. Checks if the trace endpoints are on opposite sides of this diagonal using `P_PointOnDivlineSide`.
3. Computes the intersection fraction with `P_InterceptVector`. Discards if behind origin.
4. Records the intercept as a thing intercept (`isaline = false`).

---

### `P_TraverseIntercepts`

```c
boolean P_TraverseIntercepts(traverser_t func, fixed_t maxfrac)
```

**Purpose:** Processes the `intercepts[]` array in order of increasing distance (nearest first) by calling `func` for each.

**Parameters:**
- `func` - The callback to invoke for each intercept.
- `maxfrac` - Maximum fraction to process (`FRACUNIT` for a full-length trace).

**Return value:** `true` if all intercepts were traversed; `false` if `func` returned `false`.

**Key logic:** Uses a simple selection-sort approach: in each iteration, finds the intercept with the smallest `frac`, calls `func` on it, then marks it as processed by setting its `frac` to `MAXINT`. This is O(n^2) but `n` is small (at most 128 intercepts). Stops if the nearest remaining intercept exceeds `maxfrac`.

---

### `P_PathTraverse`

```c
boolean P_PathTraverse(fixed_t x1, fixed_t y1, fixed_t x2, fixed_t y2,
                       int flags, boolean (*trav)(intercept_t*))
```

**Purpose:** The primary ray-casting function. Traces a line from `(x1, y1)` to `(x2, y2)` through the blockmap, collects all line and thing intercepts, and delivers them in order to the traversal callback `trav`.

**Parameters:**
- `x1`, `y1` - Ray start point.
- `x2`, `y2` - Ray end point.
- `flags` - Combination of `PT_ADDLINES`, `PT_ADDTHINGS`, `PT_EARLYOUT`.
- `trav` - Traversal callback; returns `false` to abort.

**Return value:** `true` if traversal completed normally; `false` if early-terminated.

**Key logic (DDA ray march):**
1. Initializes `intercept_p = intercepts` and increments `validcount`.
2. Nudges start point if it falls exactly on a blockmap grid line to avoid precision issues.
3. Sets up the global `trace` divline.
4. Converts both endpoints from world coordinates to blockmap cell coordinates `(xt1,yt1)` and `(xt2,yt2)`.
5. Computes `mapxstep`, `mapystep` (+1, -1, or 0) and the DDA step rates `xstep` and `ystep`.
6. Iterates up to 64 steps: at each cell `(mapx, mapy)`, calls `P_BlockLinesIterator` if `PT_ADDLINES` and `P_BlockThingsIterator` if `PT_ADDTHINGS`. Steps to the next cell by comparing `yintercept` and `xintercept` to decide whether to advance in X or Y (standard Bresenham/DDA logic).
7. When `(mapx, mapy) == (xt2, yt2)`, the destination cell has been processed.
8. Calls `P_TraverseIntercepts(trav, FRACUNIT)` to dispatch intercepts in order.

---

## Data Structures

### `divline_t` (defined in `p_local.h`, used throughout)

A ray or line segment: origin `(x, y)` plus direction `(dx, dy)`.

### `intercept_t` (defined in `p_local.h`, stored in `intercepts[]`)

One intersection record from a path traverse:
- `frac`: fractional distance along the trace.
- `isaline`: discriminator for the union.
- `d.line` or `d.thing`: the intersected object.

---

## Dependencies

| File | Why needed |
|------|-----------|
| `stdlib.h` | `abs()` |
| `m_bbox.h` | Box index constants |
| `doomdef.h` | Core types and constants |
| `p_local.h` | All play-local declarations, struct types, blockmap globals |
| `r_state.h` | `R_PointInSubsector`, `lines[]`, `blockmaplump`, `blockmap`, `blocklinks`, `bmapwidth`, `bmapheight`, `bmaporgx`, `bmaporgy`, `validcount` |
