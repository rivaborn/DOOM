# File Overview

**Source file:** `linuxdoom-1.10/z_zone.c`
**Module:** Zone Memory Allocator Implementation (`Z_` prefix)
**RCS revision:** `1.4  1997/02/03 16:47:58`

`z_zone.c` is the complete implementation of DOOM's custom zone memory allocator. The allocator replaces the C standard `malloc`/`free` with a deterministic, cache-friendly, arena-based system tailored to the engine's allocation patterns and memory constraints of early-1990s PC hardware.

## Design Philosophy

The zone allocator rests on three key ideas:

1. **Single contiguous arena.** The entire heap is a single block obtained from the OS at startup via `I_ZoneBase`. All engine allocations are carved out of it. There is no interaction with the C runtime heap after initialization.

2. **Purge tags instead of explicit freeing.** Every allocation carries a "purge level" tag (`PU_STATIC`, `PU_LEVEL`, `PU_CACHE`, etc.). Allocations below `PU_PURGELEVEL` (100) must be freed explicitly. Allocations at or above `PU_PURGELEVEL` are cache blocks: `Z_Malloc` can silently evict them if it needs the space, zeroing the caller's cached pointer in the process. The caller checks for `NULL` and reloads if necessary.

3. **Roving first-fit with automatic coalescing.** A roving pointer (`mainzone->rover`) remembers where the last successful allocation ended. The next search starts there and wraps around. Free blocks that become adjacent are immediately merged, so there are never two consecutive free blocks and no separate compaction pass is needed.

The allocator was noted by John Carmack as possibly the only part of the DOOM codebase worth carrying into Quake.

---

## Memory Layout

```
+------------------+---------------------------------------------------+
|   memzone_t      |          block data area                          |
|  (header/anchor) |                                                   |
+------------------+---------------------------------------------------+
^                  ^
mainzone           first memblock_t header
```

Inside the block data area, every byte belongs to exactly one `memblock_t`-prefixed region. There are no gaps:

```
+----------+--------+----------+--------+----------+--------+
| memblock | data   | memblock | data   | memblock | data   | ...
|  header  |        |  header  |        |  header  |        |
+----------+--------+----------+--------+----------+--------+
 <-- size --------> <-- size --------> <-- size ------->
```

`block->size` always includes the `memblock_t` header itself, so `(byte*)block + block->size == (byte*)block->next` is an invariant maintained throughout.

The `blocklist` field inside `memzone_t` acts as a sentinel/anchor node for the circular doubly-linked list. Its `user` pointer is set to `(void*)zone` (non-NULL, non-purgable), so it is never treated as a free or purgable block.

---

## Global Variables

| Name         | Type          | Scope  | Description |
|--------------|---------------|--------|-------------|
| `mainzone`   | `memzone_t *` | File-global (external linkage) | Pointer to the single zone that the entire engine uses. Set once by `Z_Init` and never changed. All `Z_*` functions operate on `mainzone` implicitly. |
| `rcsid`      | `const char[]`| File-static | RCS version string embedded in the binary; not used at runtime. |

---

## Internal Constants

| Name           | Value      | Description |
|----------------|------------|-------------|
| `ZONEID`       | `0x1d4a11` | Magic number stored in `memblock_t::id` for every live block. Checked by `Z_Free` and `Z_ChangeTag2` to detect corruption or double-free. Cleared to `0` on free. |
| `MINFRAGMENT`  | `64`       | Minimum size in bytes of a leftover fragment worth keeping as a separate free block after an allocation. If the excess after carving out the requested block is smaller than this, it is silently absorbed into the allocated block to avoid polluting the list with tiny fragments. |

---

## Data Structures

### `memzone_t` (internal to `z_zone.c`)

```c
typedef struct
{
    int        size;       // total bytes of the entire zone, including this header
    memblock_t blocklist;  // sentinel node anchoring the circular list
    memblock_t* rover;     // roving allocation pointer
} memzone_t;
```

The zone header lives at the very beginning of the arena. `blocklist` is a permanent, always-in-use sentinel whose `user` field points back to the zone itself (`(void*)zone`) and whose `tag` is `PU_STATIC`. This means the rover can never "lap" into the sentinel without triggering the wraparound detection in `Z_Malloc`.

`rover` is updated after each successful `Z_Malloc` to point to the block **following** the just-allocated one, so the next search starts from a fresh position rather than re-scanning already-packed memory.

### `memblock_t` (declared in `z_zone.h`, used throughout)

```c
typedef struct memblock_s
{
    int             size;   // total bytes including this header
    void**          user;   // NULL = free; ptr to owner variable if in use
    int             tag;    // PU_* purge level
    int             id;     // ZONEID magic (0x1d4a11) when live, 0 when free
    struct memblock_s* next;
    struct memblock_s* prev;
} memblock_t;
```

See `z_zone.h` documentation for the full field-by-field description.

---

## Functions

### `Z_ClearZone`

```c
void Z_ClearZone(memzone_t* zone)
```

**Purpose:** Resets a zone so it appears to contain a single large free block spanning the entire data area. Used internally by `Z_Init`.

**Parameters:**

| Parameter | Type          | Description |
|-----------|---------------|-------------|
| `zone`    | `memzone_t *` | The zone to reset. Must already have `zone->size` set correctly. |

**Return value:** `void`

**Key logic:**
- Creates a single `memblock_t` header immediately after the `memzone_t` header.
- Sets `block->user = NULL` (marks it free).
- Sets `block->size = zone->size - sizeof(memzone_t)` (the entire remaining space).
- Links the block into the circular list: `blocklist.next = blocklist.prev = block` and `block->next = block->prev = &zone->blocklist`.
- Points `zone->rover` at the new free block.
- Sets up the sentinel `blocklist` with `user = (void*)zone` and `tag = PU_STATIC` so it is never mistaken for a free or purgable block.

---

### `Z_Init`

```c
void Z_Init(void)
```

**Purpose:** Bootstraps the zone allocator. Called once at engine startup, before any other `Z_*` calls.

**Parameters:** None.

**Return value:** `void`

**Key logic:**
- Calls `I_ZoneBase(&size)` to obtain a large raw memory block from the OS. `size` is set to the actual number of bytes returned.
- Casts the returned pointer to `memzone_t*` and assigns it to `mainzone`.
- Sets `mainzone->size = size`.
- Manually mirrors the `Z_ClearZone` setup (initializing sentinel, single free block, rover) rather than calling `Z_ClearZone`, for clarity.
- After this call, `mainzone` holds a single free block of `size - sizeof(memzone_t)` bytes.

---

### `Z_Free`

```c
void Z_Free(void* ptr)
```

**Purpose:** Frees a previously allocated zone block. Performs coalescing with adjacent free blocks to prevent fragmentation.

**Parameters:**

| Parameter | Type     | Description |
|-----------|----------|-------------|
| `ptr`     | `void *` | Pointer to the data area of a live block (i.e., the value returned by `Z_Malloc`). Must not be `NULL`. |

**Return value:** `void`

**Key logic:**

1. **Recover header.** `block = (memblock_t*)((byte*)ptr - sizeof(memblock_t))`. This is always valid because `Z_Malloc` always returns `(byte*)base + sizeof(memblock_t)`.

2. **Validate magic.** If `block->id != ZONEID`, calls `I_Error("Z_Free: freed a pointer without ZONEID")`. This catches wild pointers, double-frees, and memory smashers.

3. **Notify owner.** If `block->user > (void**)0x100` (i.e., it is a real pointer, not the sentinel value `2`), sets `*block->user = 0` so the caller's cached pointer becomes `NULL`. This is the crucial mechanism that makes cache eviction safe: callers always check for `NULL` before using a purgable pointer.

4. **Mark free.** Sets `block->user = NULL`, `block->tag = 0`, `block->id = 0`.

5. **Coalesce backward.** If `block->prev->user == NULL` (previous block is free), merges them: `prev->size += block->size`, `prev->next = block->next`, fixes back-link. If the rover pointed at `block`, updates rover to the merged block.

6. **Coalesce forward.** If `block->next->user == NULL` (next block is free), merges: `block->size += next->size`, `block->next = next->next`, fixes back-link. If the rover pointed at `next`, updates rover to `block`.

After `Z_Free` there are never two consecutive free blocks in the list. No external compaction pass is ever required.

---

### `Z_Malloc`

```c
void* Z_Malloc(int size, int tag, void* user)
```

**Purpose:** Allocates `size` bytes from the zone with the given purge level. This is the core allocator function and the most complex piece of `z_zone.c`.

**Parameters:**

| Parameter | Type     | Description |
|-----------|----------|-------------|
| `size`    | `int`    | Number of data bytes requested. Must be positive. |
| `tag`     | `int`    | Purge level (`PU_*` constant). Determines when the block may be automatically evicted. |
| `user`    | `void *` | Pointer to the caller's pointer variable (e.g., `&mytexture`). The allocator stores this and writes the data address back through it. May be `NULL` only if `tag < PU_PURGELEVEL`. |

**Return value:** `void *` — pointer to the usable data area (immediately after the `memblock_t` header). Never returns `NULL`; calls `I_Error` if allocation fails.

**Key logic — step by step:**

1. **Alignment.** `size = (size + 3) & ~3` — rounds up to the next 4-byte boundary.

2. **Add header overhead.** `size += sizeof(memblock_t)` — the search is for a block that can hold both the header and the data.

3. **Position rover.** `base = mainzone->rover`. If the block just before the rover is free, back up one to start from a potentially usable position.

4. **Circular first-fit scan.** Iterates forward through the block list starting at `base`, using `rover` as the lookahead pointer:

   - If `rover == start` (we have gone all the way around the list without finding a suitable block): calls `I_Error("Z_Malloc: failed on allocation of %i bytes", size)`.

   - If `rover->user != NULL` (block is in use):
     - If `rover->tag < PU_PURGELEVEL`: the block cannot be evicted. Advance both `base` and `rover` past it and continue.
     - If `rover->tag >= PU_PURGELEVEL`: the block is a cache entry and can be stolen. Back `base` up one step (so coalescing in `Z_Free` will merge the freed space into `base` if `base` is also free), call `Z_Free(rover+1)`, then re-advance `base` and `rover`.

   - If `rover->user == NULL`: the block is free; advance `rover` to the next block.

   - The loop condition `while (base->user || base->size < size)` keeps going until `base` is both free and large enough.

5. **Fragment.** `extra = base->size - size`. If `extra > MINFRAGMENT` (64 bytes), a new free `memblock_t` is carved out at `(byte*)base + size` with `size = extra`, linked in between `base` and `base->next`. `base->size` is then set to `size` exactly.

6. **Assign ownership.**
   - If `user != NULL`: `base->user = user; *(void**)user = (byte*)base + sizeof(memblock_t)`. The caller's pointer now points to the data; the block header knows who owns it.
   - If `user == NULL` (only valid for non-purgable tags): `base->user = (void*)2`. The value `2` is a non-`NULL` sentinel (above the `0x100` threshold check in `Z_Free`) that means "in use but no owner pointer to clear".

7. **Finalize.** `base->tag = tag; base->id = ZONEID; mainzone->rover = base->next`.

8. **Return** `(byte*)base + sizeof(memblock_t)`.

**Cache eviction summary:** When `Z_Malloc` evicts a purgable block during the scan, it calls `Z_Free` on it. `Z_Free` zeroes `*block->user`, so the subsystem that owned the cache entry (e.g., `R_GenerateComposite` for textures) will find its pointer is `NULL` the next time it checks, and will regenerate the data. This is a form of implicit garbage collection with demand-driven regeneration — the cache is never explicitly invalidated, it just disappears when memory is tight.

---

### `Z_FreeTags`

```c
void Z_FreeTags(int lowtag, int hightag)
```

**Purpose:** Bulk-frees all live blocks whose purge tag is in the inclusive range `[lowtag, hightag]`. The primary use is end-of-level cleanup.

**Parameters:**

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `lowtag`  | `int` | Lower bound of the tag range to free (inclusive). |
| `hightag` | `int` | Upper bound of the tag range to free (inclusive). |

**Return value:** `void`

**Key logic:**
- Iterates the entire block list (skipping the sentinel).
- Saves `next = block->next` **before** calling `Z_Free`, since `Z_Free` modifies the list linkage (and may merge blocks, invalidating the old `next`).
- Skips blocks with `block->user == NULL` (already free).
- Calls `Z_Free((byte*)block + sizeof(memblock_t))` for matching blocks.

**Typical call sites:**

| Call                                        | Where             | Purpose |
|---------------------------------------------|-------------------|---------|
| `Z_FreeTags(PU_LEVEL, PU_PURGELEVEL-1)`     | `G_DoLoadLevel`   | Discard all level-specific allocations before loading a new level. |
| `Z_FreeTags(PU_PURGELEVEL, 0x7fffffff)`     | (not called directly) | Would discard all cache; `Z_Malloc` instead evicts selectively. |

---

### `Z_DumpHeap`

```c
void Z_DumpHeap(int lowtag, int hightag)
```

**Purpose:** Prints a human-readable diagnostic dump of all blocks in the given tag range to `stdout`. Also checks three structural invariants for every block in the list (regardless of tag range) and prints `ERROR:` lines for violations.

**Parameters:**

| Parameter | Type  | Description |
|-----------|-------|-------------|
| `lowtag`  | `int` | Lower bound of tags to print. |
| `hightag` | `int` | Upper bound of tags to print. |

**Return value:** `void`

**Invariants checked (non-fatal, printed as strings):**
- `(byte*)block + block->size == (byte*)block->next` — blocks are contiguous.
- `block->next->prev == block` — doubly-linked list is consistent.
- `!(!block->user && !block->next->user)` — no two consecutive free blocks.

---

### `Z_FileDumpHeap`

```c
void Z_FileDumpHeap(FILE* f)
```

**Purpose:** Same structural dump as `Z_DumpHeap` but writes to an arbitrary `FILE*` rather than `stdout`, and dumps all blocks without tag filtering. Suitable for writing to a debug log file.

**Parameters:**

| Parameter | Type     | Description |
|-----------|----------|-------------|
| `f`       | `FILE *` | Open, writable file handle for the output. |

**Return value:** `void`

---

### `Z_CheckHeap`

```c
void Z_CheckHeap(void)
```

**Purpose:** Validates zone integrity and calls `I_Error` (fatal abort) on any violation. Used in debug builds to catch corruption early.

**Parameters:** None.

**Return value:** `void`

**Invariants checked (fatal via `I_Error`):**
1. Block size-touch: `(byte*)block + block->size != (byte*)block->next`
2. Back-link consistency: `block->next->prev != block`
3. No consecutive free blocks: `!block->user && !block->next->user`

---

### `Z_ChangeTag2`

```c
void Z_ChangeTag2(void* ptr, int tag)
```

**Purpose:** Changes the purge tag of a live block in place. The `Z_ChangeTag` macro (from `z_zone.h`) should be used by callers; it performs a preliminary `ZONEID` check with the call-site file/line before delegating here.

**Parameters:**

| Parameter | Type     | Description |
|-----------|----------|-------------|
| `ptr`     | `void *` | Data pointer (as returned by `Z_Malloc`). |
| `tag`     | `int`    | New purge level to assign. |

**Return value:** `void`

**Key logic:**
- Recovers `block` from `ptr` by subtracting `sizeof(memblock_t)`.
- Checks `block->id == ZONEID`; calls `I_Error` if not.
- Checks that if `tag >= PU_PURGELEVEL` then `block->user >= 0x100` (a real owner pointer exists), because purgable blocks must be able to null the owner's reference on eviction.
- Sets `block->tag = tag`.

**Common usage patterns:**

```c
// Load a texture into cache; can be evicted later
data = Z_Malloc(size, PU_STATIC, &data);
/* ... use data ... */
Z_ChangeTag(data, PU_CACHE);  // Now evictable; caller must check for NULL before next use.

// Promote a cache block back to permanent
Z_ChangeTag(data, PU_STATIC); // Will not be evicted again.
```

---

### `Z_FreeMemory`

```c
int Z_FreeMemory(void)
```

**Purpose:** Returns a conservative estimate of available bytes: the sum of all currently free blocks plus all purgable blocks (since the latter can be evicted on demand by `Z_Malloc`).

**Parameters:** None.

**Return value:** `int` — approximate free bytes.

**Key logic:**
- Iterates the full block list.
- Adds `block->size` to `free` for any block where `block->user == NULL` (free) or `block->tag >= PU_PURGELEVEL` (purgable/cache).
- Does NOT account for the `memblock_t` header overhead of purgable blocks that would be reclaimed as free space after eviction, so the result is a slight over-estimate.

---

## The Cache Eviction System in Detail

The most sophisticated aspect of `z_zone.c` is the interplay between `PU_CACHE` blocks and `Z_Malloc`'s rover loop. The mechanism works as follows:

### Allocation-time eviction

When `Z_Malloc` needs space and the rover encounters a block with `tag >= PU_PURGELEVEL`, it calls `Z_Free` on it. `Z_Free` sets `*(block->user) = NULL`, informing the owner that its cached copy is gone. The freed space is immediately coalesced and becomes available to satisfy the current `Z_Malloc` request.

### Owner-side protocol

Any subsystem that uses `PU_CACHE` must follow this pattern:

```c
// Texture cache example (from r_data.c)
if (!texturecache[texnum])
{
    // Cache miss: generate and store
    texturecache[texnum] = Z_Malloc(size, PU_CACHE, &texturecache[texnum]);
    /* ... fill in data ... */
}
// texturecache[texnum] is now valid (either was already, or just reloaded)
```

The `&texturecache[texnum]` passed as `user` is the critical link: when the zone evicts the block, it stores `NULL` back into `texturecache[texnum]`, and the next access finds `NULL` and regenerates.

### Why this is cache-friendly

- No separate cache-invalidation calls are needed.
- The zone never needs to "scan for purgables" in a separate pass: eviction is opportunistic, happening exactly when allocation pressure demands it.
- By starting the rover just behind the new allocation, recently-allocated blocks are unlikely to be immediately evicted again, providing a mild LRU-like approximation.

---

## Dependencies

| Module        | Symbol(s) Used | Description |
|---------------|----------------|-------------|
| `z_zone.h`    | `memblock_t`, `PU_*` constants, function declarations | Own public header. |
| `i_system.h`  | `I_ZoneBase`, `I_Error` | `I_ZoneBase(&size)` obtains the raw memory arena. `I_Error` is called on unrecoverable allocation failures and integrity violations. |
| `doomdef.h`   | `byte` | Provides the `unsigned char` typedef used extensively for pointer arithmetic. |
