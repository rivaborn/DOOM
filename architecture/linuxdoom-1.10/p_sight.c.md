# File Overview

`p_sight.c` implements the Line-of-Sight (LOS) check used throughout DOOM's AI system. The primary exported function, `P_CheckSight`, determines whether a straight, unobstructed line exists between two map objects — for example, whether a monster can see the player or whether a monster can target another actor.

The algorithm has two stages:
1. **REJECT table fast rejection**: A precomputed bit matrix in the WAD file allows instant rejection of pairs of sectors that were determined by the level designer's tools to be mutually invisible. If both objects' sectors are marked as mutually invisible in the REJECT table, the check returns false immediately without any geometry traversal.
2. **BSP tree traversal with slope tracking**: If the REJECT table does not rule out LOS, the algorithm traverses the BSP tree from the root, tracking a vertical "sight window" (expressed as top and bottom slopes from the looker's eye height). Each two-sided wall crossing narrows this window. If it ever becomes degenerate (bottom slope >= top slope), sight is blocked.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `fixed_t` | `sightzstart` | Z coordinate of the eye of the looker (t1->z + t1->height - t1->height/4). Computed at the start of each check. |
| `fixed_t` | `topslope` | Current upper bound of the vertical sight window. Initialized to the top of t2, narrowed by two-sided walls. |
| `fixed_t` | `bottomslope` | Current lower bound of the vertical sight window. Initialized to the bottom of t2, narrowed by two-sided walls. |
| `divline_t` | `strace` | The sight trace: origin at t1 (x, y) and delta (dx, dy) pointing toward t2. Used as a 2D line for crossing tests. |
| `fixed_t` | `t2x` | X coordinate of the target (t2->x). |
| `fixed_t` | `t2y` | Y coordinate of the target (t2->y). |
| `int[2]` | `sightcounts` | Debug/profiling counters: `[0]` counts REJECT-table rejections, `[1]` counts full BSP traversals. |

## Functions

### `P_DivlineSide`
```c
int P_DivlineSide(fixed_t x, fixed_t y, divline_t* node)
```
**Purpose:** Determines on which side of a dividing line a point lies. Returns 0 for front, 1 for back, 2 for on the line.

**Parameters:**
- `x`, `y` - The point to test.
- `node` - The dividing line (origin + delta).

**Return value:** `0` (front/left), `1` (back/right), or `2` (collinear).

**Key logic:** Handles the degenerate cases of vertical (`dx == 0`) and horizontal (`dy == 0`) divlines with simple comparisons. For general lines, computes the cross product `(node->dy * dx) - (dy * node->dx)` using right-shifted values to avoid fixed-point overflow.

### `P_InterceptVector2`
```c
fixed_t P_InterceptVector2(divline_t* v2, divline_t* v1)
```
**Purpose:** Computes the fractional intersection point along `v2` where `v1` crosses it. Returns the parametric `t` in `[0, FRACUNIT]` where the intersection occurs along `v2`.

**Parameters:**
- `v2` - The line along which to measure the fractional position.
- `v1` - The crossing line.

**Return value:** A `fixed_t` fraction in `[0, FRACUNIT]` representing how far along `v2` the intersection point lies. Returns 0 if the lines are parallel.

**Key logic:** Uses the standard 2D line intersection formula with shifted intermediate values to mitigate fixed-point overflow. The `>>8` shifts are a precision tradeoff specific to DOOM's 16.16 fixed-point arithmetic.

### `P_CrossSubsector`
```c
boolean P_CrossSubsector(int num)
```
**Purpose:** Tests whether the sight trace (`strace`) can cross a given BSP leaf (subsector) without being blocked.

**Parameters:**
- `num` - Index into the `subsectors` array.

**Return value:** `true` if the trace passes through the subsector unobstructed; `false` if a wall blocks it.

**Key logic:**
- Iterates all segs in the subsector.
- For each seg's linedef, checks `validcount` to skip lines already checked in this traversal.
- Tests whether both vertices of the line are on the same side of `strace` (if so, `strace` does not cross this line).
- Tests whether both endpoints of `strace` are on the same side of the line (if so, `strace` does not cross this line).
- For crossed two-sided lines: computes the `opentop` (min of both ceiling heights) and `openbottom` (max of both floor heights). If `openbottom >= opentop`, the opening is closed — sight is blocked.
- Computes the fractional intersection distance `frac` and updates `topslope` / `bottomslope` if the floor/ceiling height differences tighten the window.
- Returns `false` if `topslope <= bottomslope` at any point (window degenerated).

### `P_CrossBSPNode`
```c
boolean P_CrossBSPNode(int bspnum)
```
**Purpose:** Recursively traverses the BSP tree, calling `P_CrossSubsector` at leaves.

**Parameters:**
- `bspnum` - BSP node index, or (with `NF_SUBSECTOR` bit set) a subsector index.

**Return value:** `true` if the trace can reach the other end unobstructed; `false` if blocked.

**Key logic:**
- If the `NF_SUBSECTOR` bit is set, calls `P_CrossSubsector` directly.
- Otherwise determines which side of the node's partition line `strace.x/y` lies on.
- Recursively crosses the near side first.
- If the target point is on the same side as the origin, the trace does not cross to the far side and can return `true`.
- Otherwise recursively crosses the far side as well.

### `P_CheckSight`
```c
boolean P_CheckSight(mobj_t* t1, mobj_t* t2)
```
**Purpose:** The main API function. Returns `true` if t1 can see t2 (no solid geometry blocks the line of sight).

**Parameters:**
- `t1` - The observer (looker).
- `t2` - The target.

**Return value:** `true` if LOS is clear, `false` if blocked or trivially rejected.

**Key logic:**
1. Computes the sector pair indices and looks up the REJECT table byte and bit for the pair.
2. If the REJECT bit is set, immediately returns `false` (early out, no geometry needed).
3. Otherwise increments `validcount` (to mark lines checked in this traversal as visited).
4. Sets up `sightzstart` as 3/4 of t1's height above its feet (eye level approximation).
5. Sets `topslope` and `bottomslope` as the angular extent from `sightzstart` to the top and bottom of t2.
6. Initializes `strace` from t1's x/y to t2's x/y.
7. Calls `P_CrossBSPNode(numnodes-1)` to traverse from the BSP root.

## Dependencies

| File | Reason |
|------|--------|
| `doomdef.h` | `NF_SUBSECTOR`, fixed-point constants |
| `i_system.h` | `I_Error` for range checking |
| `p_local.h` | `divline_t`, mobj definitions, `validcount` |
| `r_state.h` | `nodes`, `sectors`, `subsectors`, `segs`, `rejectmatrix`, `numnodes`, `numsubsectors`, `numsectors` |
