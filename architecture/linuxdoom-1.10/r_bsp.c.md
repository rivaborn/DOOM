# File Overview

`r_bsp.c` is the BSP (Binary Space Partitioning) traversal engine at the heart of DOOM's software renderer. It implements the front-to-back rendering order that allows DOOM to correctly draw an arbitrary 3D scene without a Z-buffer.

## Role in the Rendering Pipeline

DOOM's maps are pre-partitioned into a BSP tree by the level compiler (DOOMBSP). Each internal node of the tree contains a partition line that divides the world into front and back halves. Leaf nodes are convex subsectors (groups of line segments bounding a single sector area).

`r_bsp.c` performs these critical tasks each frame:

1. **BSP traversal** (`R_RenderBSPNode`): Walks the tree recursively, always visiting the half containing the viewpoint first. This guarantees front-to-back ordering.
2. **Subsector rendering** (`R_Subsector`): For each leaf subsector reached during traversal, finds or creates visplane records for the floor and ceiling, adds sprites for things in the sector, and calls `R_AddLine` on each wall segment.
3. **Segment clipping** (`R_AddLine`, `R_ClipSolidWallSegment`, `R_ClipPassWallSegment`): As walls are submitted front-to-back, a clipping list (`solidsegs`) tracks which horizontal screen columns are already fully occluded by solid walls. New segments are clipped against this list. Once a column is blocked by a solid wall, nothing behind it can be visible — this is the key BSP optimization.
4. **Bounding box rejection** (`R_CheckBBox`): Before recursing into the back half of a BSP node, checks whether the node's bounding box projects onto any open (unoccluded) screen columns. If not, the entire subtree is skipped.

## Global Variables

| Type | Name | Description |
|---|---|---|
| `seg_t*` | `curline` | Pointer to the line segment currently being processed by `R_AddLine`. Set on entry to `R_AddLine`; read by `R_StoreWallRange` in `r_segs.c`. |
| `side_t*` | `sidedef` | The sidedef of the current segment. Set in `R_StoreWallRange`; used throughout wall rendering. |
| `line_t*` | `linedef` | The linedef of the current segment. Set in `R_StoreWallRange`. |
| `sector_t*` | `frontsector` | The sector on the viewer's side of the current segment. Set in `R_Subsector` and `R_AddLine`. |
| `sector_t*` | `backsector` | The sector on the far side of the current segment (NULL for one-sided walls). Set in `R_AddLine`. |
| `drawseg_t` | `drawsegs[MAXDRAWSEGS]` | Array of up to 256 `drawseg_t` records, one per visible wall range stored this frame. |
| `drawseg_t*` | `ds_p` | Pointer to the next free slot in `drawsegs`. Incremented by `R_StoreWallRange`; compared against `drawsegs` by `R_DrawSprite` in `r_things.c` for sprite clipping. |
| `cliprange_t*` | `newend` | One past the last valid entry in `solidsegs`. Maintained as clipping ranges merge and expand. |
| `cliprange_t` | `solidsegs[MAXSEGS]` | Array of up to 32 clipping ranges. Each entry defines a horizontal screen column interval that is completely occluded by a solid (one-sided) wall. Bounded by sentinel entries at `[-infinity, -1]` and `[viewwidth, +infinity]`. |
| `int` | `checkcoord[12][4]` | Static lookup table mapping a viewer-relative position code (3x4 grid representing which of the 9 spatial quadrants around a bounding box the viewer is in) to the two bounding-box corners that define the extreme view angles to the box. Used by `R_CheckBBox`. |

## Functions

### `R_ClearDrawSegs`

```c
void R_ClearDrawSegs(void)
```

**Purpose:** Resets the draw segment pointer to the beginning of the `drawsegs` array, discarding all segments from the previous frame.

**Parameters:** None.

**Return Value:** `void`

---

### `R_ClearClipSegs`

```c
void R_ClearClipSegs(void)
```

**Purpose:** Resets the solid-segment clip list to its initial state: two sentinel entries that represent "screen left is blocked at -infinity to -1" and "screen right is blocked from viewwidth to +infinity". All screen columns are initially open.

**Parameters:** None.

**Return Value:** `void`

**Key Logic:** Sets `solidsegs[0] = {-0x7fffffff, -1}` and `solidsegs[1] = {viewwidth, 0x7fffffff}`. Sets `newend = solidsegs + 2`.

---

### `R_ClipSolidWallSegment`

```c
void R_ClipSolidWallSegment(int first, int last)
```

**Purpose:** Records a solid (one-sided) wall segment covering screen columns `[first, last]`. Draws any visible portions via `R_StoreWallRange` and merges the range into `solidsegs`, expanding or merging existing ranges as needed.

**Parameters:**
- `first` (`int`): First screen column of the segment.
- `last` (`int`): Last screen column of the segment.

**Return Value:** `void`

**Key Logic:**

This is the core occlusion algorithm. The `solidsegs` array is a sorted list of non-overlapping ranges of fully-occluded columns.

1. Find the first clip range `start` that overlaps or is adjacent to `[first, last]`.
2. If the new segment's left edge is before `start->first`: there is a visible gap. Draw it with `R_StoreWallRange` and optionally extend `start->first` left.
3. If the segment extends past `start->last`, continue checking subsequent ranges for more gaps, drawing each gap, and ultimately merging all overlapping ranges into one.
4. After drawing, consolidate the `solidsegs` array by removing any ranges that have been subsumed by the newly expanded range (the `crunch:` label).

This means that over the course of rendering a frame, `solidsegs` grows from 2 entries (just sentinels) up to a maximum of `MAXSEGS` (32), and wall segments behind fully-occluded columns are never drawn.

---

### `R_ClipPassWallSegment`

```c
void R_ClipPassWallSegment(int first, int last)
```

**Purpose:** Clips a transparent/two-sided wall segment (a "window" — a linedef with upper/lower textures) against the solid-segment list and draws visible portions via `R_StoreWallRange`. Unlike `R_ClipSolidWallSegment`, this does **not** add the range to `solidsegs`, because two-sided windows do not fully occlude the view.

**Parameters:**
- `first` (`int`): First screen column.
- `last` (`int`): Last screen column.

**Return Value:** `void`

**Key Logic:** Walks `solidsegs` to find gaps between solid ranges, drawing each visible portion with `R_StoreWallRange`. Does not modify `solidsegs`.

---

### `R_AddLine`

```c
void R_AddLine(seg_t* line)
```

**Purpose:** Processes a single line segment from a subsector during BSP traversal. Performs angle-based view frustum clipping, determines whether the segment is solid or passable, and dispatches to the appropriate clip function.

**Parameters:**
- `line` (`seg_t*`): The line segment to consider.

**Return Value:** `void`

**Key Logic:**

1. Sets `curline = line`.
2. Computes view angles to both endpoints using `R_PointToAngle`.
3. **Backface culling:** If `angle1 - angle2 >= ANG180`, the segment faces away from the viewer — skip it.
4. **Frustum clipping:** Clips the endpoint angles against `clipangle` (half the field of view). If both endpoints are off one side, skip.
5. **Screen-space projection:** Converts clipped angles to screen X coordinates via `viewangletox[]`. If `x1 == x2` (zero-width), skip.
6. Sets `backsector`.
7. **Routing:**
   - No backsector (one-sided wall): `goto clipsolid` → `R_ClipSolidWallSegment`.
   - Closed door (back floor above front ceiling or vice versa): `goto clipsolid`.
   - Height difference (window): `goto clippass` → `R_ClipPassWallSegment`.
   - Identical geometry and same flat/light with no midtexture: skip entirely (invisible trigger lines).

---

### `R_CheckBBox`

```c
boolean R_CheckBBox(fixed_t* bspcoord)
```

**Purpose:** Fast visibility test for a BSP node's bounding box. Returns `true` if any part of the box might project onto an open (non-occluded) screen column; returns `false` if the box is entirely behind solid walls or outside the view frustum, allowing the subtree to be culled.

**Parameters:**
- `bspcoord` (`fixed_t*`): A 4-element array `[BOXTOP, BOXBOTTOM, BOXLEFT, BOXRIGHT]` defining the bounding box in world coordinates.

**Return Value:** `boolean` — `true` if the subtree might be visible, `false` if it can be safely skipped.

**Key Logic:**

1. Determines the viewer's quadrant relative to the box using `viewx`/`viewy` vs. box edges, yielding a 3x3 position code (`boxpos`).
2. Uses the `checkcoord[12][4]` table to select the two extreme corners of the box that form the widest angular span from the viewpoint.
3. Computes angles to those corners and clips against the view frustum.
4. Converts angles to screen X range `[sx1, sx2]`.
5. Checks whether the entire screen range `[sx1, sx2]` is already covered by a single entry in `solidsegs`. If yes, the box is fully occluded → return `false`. Otherwise → return `true`.

---

### `R_Subsector`

```c
void R_Subsector(int num)
```

**Purpose:** Processes a BSP leaf (subsector). Establishes floor/ceiling visplanes, adds sprites for things in the sector, and feeds each line segment to `R_AddLine`.

**Parameters:**
- `num` (`int`): Index into the `subsectors[]` array.

**Return Value:** `void`

**Key Logic:**

1. Looks up `sub = &subsectors[num]` and sets `frontsector = sub->sector`.
2. **Floor plane:** If the sector floor is below the viewpoint (`floorheight < viewz`), calls `R_FindPlane` to find or create a `visplane_t` for the floor. Otherwise sets `floorplane = NULL`.
3. **Ceiling plane:** If the ceiling is above the viewpoint or is a sky flat, calls `R_FindPlane` for the ceiling. Otherwise `ceilingplane = NULL`.
4. Calls `R_AddSprites(frontsector)` to project sprites of all things in the sector onto the screen.
5. Iterates over all `sub->numlines` segments and calls `R_AddLine` on each.

---

### `R_RenderBSPNode`

```c
void R_RenderBSPNode(int bspnum)
```

**Purpose:** Recursively traverses the BSP tree, rendering subsectors in strict front-to-back order.

**Parameters:**
- `bspnum` (`int`): Index of the current BSP node. If the high bit (`NF_SUBSECTOR`) is set, this is a subsector index.

**Return Value:** `void`

**Key Logic:**

1. **Leaf test:** If `bspnum & NF_SUBSECTOR`, call `R_Subsector(bspnum & ~NF_SUBSECTOR)` and return.
2. Look up `bsp = &nodes[bspnum]`.
3. Determine `side = R_PointOnSide(viewx, viewy, bsp)` — which half-space the viewer is in (0 = front, 1 = back).
4. **Recurse front:** `R_RenderBSPNode(bsp->children[side])` — the viewer's side is always rendered first.
5. **Back-side test:** Call `R_CheckBBox(bsp->bbox[side^1])`. Only recurse into the back half (`R_RenderBSPNode(bsp->children[side^1])`) if the bounding box might still be visible.

This is the classic front-to-back BSP traversal. Because solid walls add entries to `solidsegs` as they are encountered, subsequent `R_CheckBBox` calls become increasingly likely to return `false`, pruning large parts of the tree.

## Data Structures

### `cliprange_t` (local to this file)

```c
typedef struct {
    int first;
    int last;
} cliprange_t;
```

Represents a contiguous range of screen columns `[first, last]` that is fully occluded by a solid wall. Used internally in `solidsegs`.

## Dependencies

| File | Reason |
|---|---|
| `doomdef.h` | Core types and constants |
| `m_bbox.h` | `BOXLEFT`, `BOXRIGHT`, `BOXTOP`, `BOXBOTTOM` constants for bounding box indexing |
| `i_system.h` | `I_Error()` for range-check error reporting |
| `r_main.h` | `viewx`, `viewy`, `viewangle`, `clipangle`, `viewangletox[]`, `R_PointToAngle()`, `R_PointOnSide()`, `rw_angle1`, `validcount` |
| `r_plane.h` | `R_FindPlane()`, `floorplane`, `ceilingplane` |
| `r_things.h` | `R_AddSprites()` |
| `doomstat.h` | `sscount` (subsector count profiling) |
| `r_state.h` | `subsectors[]`, `numsubsectors`, `segs[]`, `nodes[]`, `numnodes`, `viewwidth` |
