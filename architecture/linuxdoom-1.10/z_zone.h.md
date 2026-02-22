# File Overview

**Source file:** `linuxdoom-1.10/z_zone.h`
**Module:** Zone Memory Allocator Public Interface (`Z_` prefix)

`z_zone.h` is the public header for DOOM's custom zone memory allocator. It exposes the full API, defines the purge-tag constants that govern block lifetimes, declares the `memblock_t` structure that forms the building block of the allocator, and provides the `Z_ChangeTag` convenience macro.

Every subsystem in the DOOM engine that allocates memory at runtime goes through this interface rather than calling `malloc`/`free` directly. The allocator operates on a single large contiguous memory arena obtained from the OS at startup (`I_ZoneBase`), and manages it entirely with an intrusive doubly-linked list of `memblock_t` headers.

John Carmack noted (per the header comment) that this was the only piece of the DOOM engine he thought might have been genuinely useful to carry forward into Quake.

---

## Data Structures

### `memblock_t`

```c
typedef struct memblock_s
{
    int            size;   // total bytes of this block, including this header
    void**         user;   // NULL = free block; non-NULL = pointer to owning pointer
    int            tag;    // purge level (PU_* constant)
    int            id;     // magic number: must equal ZONEID (0x1d4a11) when live
    struct memblock_s* next;
    struct memblock_s* prev;
} memblock_t;
```

Each allocated (or free) region in the zone is prefixed by one of these 24-byte (on 32-bit) headers. Together they form a circular doubly-linked list anchored at `mainzone->blocklist`.

| Field  | Type              | Description |
|--------|-------------------|-------------|
| `size` | `int`             | Total byte count of this block **including** the `memblock_t` header itself. Because blocks are always contiguous, `(byte*)block + block->size` is exactly the address of the next header. |
| `user` | `void**`          | When non-`NULL` and `> 0x100`, points to the caller's pointer variable. On allocation, `Z_Malloc` writes the data pointer back through `*user`. When the block is freed or evicted, `*user` is zeroed so dangling references become `NULL`. A value of exactly `2` means "in use but has no owner pointer" (used for `PU_STATIC` blocks below `PU_PURGELEVEL` that were allocated with a `NULL` user). `NULL` means the block is currently free. |
| `tag`  | `int`             | The purge level; one of the `PU_*` constants. Determines when the allocator is allowed to reclaim this block automatically. |
| `id`   | `int`             | Set to `ZONEID` (0x1d4a11) when the block is live; cleared to `0` when freed. Used by `Z_Free` and `Z_ChangeTag2` to detect double-frees or wild pointer corruption. |
| `next` | `memblock_t *`    | Next block in the circular list. |
| `prev` | `memblock_t *`    | Previous block in the circular list. |

---

## Global Variables

Defined in `z_zone.c`, not re-exported in this header, but referenced through the macro:

| Name       | Type          | Description |
|------------|---------------|-------------|
| `mainzone` | `memzone_t *` | The single global zone. All allocation/free operations act on this zone. Initialized in `Z_Init`. |

---

## Purge Tag Constants

Tags control the lifetime of an allocation. The allocator divides tags into two categories at the boundary `PU_PURGELEVEL`:

```
Tags < PU_PURGELEVEL (< 100):  block will NOT be automatically evicted
Tags >= PU_PURGELEVEL (>= 100): block CAN be evicted automatically when Z_Malloc needs space
```

| Constant        | Value | Description |
|-----------------|-------|-------------|
| `PU_STATIC`     | `1`   | Permanent for the entire execution lifetime of the engine. Never freed except by an explicit `Z_Free` call. Used for the zone sentinel block itself, WAD lump directory, core data structures, etc. |
| `PU_SOUND`      | `2`   | Static while a sound is playing. Freed when the sound finishes. |
| `PU_MUSIC`      | `3`   | Static while music is playing. Freed when the music stops. |
| `PU_DAVE`       | `4`   | A catch-all "static" tag reserved for anything else that Dave Taylor (credited by name in the comment) needed to keep live for an indefinite period. |
| `PU_LEVEL`      | `50`  | Lives for the duration of one level. Freed en masse by `Z_FreeTags(PU_LEVEL, PU_PURGELEVEL-1)` when the level ends. Texture composites, map geometry, thinker structures, etc. use this tag. |
| `PU_LEVSPEC`    | `51`  | Same lifetime as `PU_LEVEL`; specifically used for "special thinker" objects within a level (moving floors, doors, etc.). |
| `PU_PURGELEVEL` | `100` | The boundary value. Blocks with `tag >= PU_PURGELEVEL` are candidates for automatic eviction during `Z_Malloc` when the rover cannot find a large-enough free block by other means. |
| `PU_CACHE`      | `101` | Cached data that can be silently discarded whenever memory pressure demands. Used for decoded/composited textures and flats that are expensive to regenerate but can be reloaded from the WAD if needed. This is the primary cache-eviction mechanism: calling code checks `if (ptr == NULL)` and reloads when needed. |

---

## Functions

### `Z_Init`

```c
void Z_Init(void);
```

Initializes the zone allocator. Calls `I_ZoneBase` to obtain a large contiguous memory block from the OS, stores it as `mainzone`, and sets up the initial state as a single large free block with the zone header acting as the list sentinel. Called once at engine startup.

---

### `Z_Malloc`

```c
void* Z_Malloc(int size, int tag, void *ptr);
```

Allocates `size` bytes from the zone with the given purge tag. `ptr` is a pointer to the caller's pointer variable (e.g., `&mytexture`); the allocator writes the new data address back through it and stores `ptr` in the block header so future eviction can null the caller's pointer automatically. Pass `NULL` for `ptr` only if `tag < PU_PURGELEVEL`.

---

### `Z_Free`

```c
void Z_Free(void *ptr);
```

Frees a previously allocated block. Validates the `ZONEID` magic, nulls the owner's pointer (if any), marks the block free, and coalesces it with immediately adjacent free blocks to prevent fragmentation.

---

### `Z_FreeTags`

```c
void Z_FreeTags(int lowtag, int hightag);
```

Iterates the entire block list and frees every live block whose tag falls in `[lowtag, hightag]`. The canonical call at level exit is `Z_FreeTags(PU_LEVEL, PU_PURGELEVEL-1)`.

---

### `Z_DumpHeap`

```c
void Z_DumpHeap(int lowtag, int hightag);
```

Prints a diagnostic dump of all blocks in the given tag range to `stdout`, together with structural integrity checks (size-touch, back-link validity, no consecutive free blocks).

---

### `Z_FileDumpHeap`

```c
void Z_FileDumpHeap(FILE *f);
```

Same as `Z_DumpHeap` but writes to an arbitrary `FILE *` (e.g., a debug log file). Dumps all blocks regardless of tag.

---

### `Z_CheckHeap`

```c
void Z_CheckHeap(void);
```

Validates the structural integrity of the entire zone heap. Calls `I_Error` on any of three detected invariant violations: block size does not touch next block, next block has wrong back-link, or two consecutive free blocks exist. Used during debugging builds.

---

### `Z_ChangeTag2`

```c
void Z_ChangeTag2(void *ptr, int tag);
```

Changes the purge tag of a live block. Validates `ZONEID` and ensures that purgable tags (`>= PU_PURGELEVEL`) are only assigned to blocks that have an owner pointer. Not called directly; use the `Z_ChangeTag` macro instead.

---

### `Z_FreeMemory`

```c
int Z_FreeMemory(void);
```

Returns the total number of free bytes currently available in the zone (sum of all free blocks plus all purgable blocks, since purgable blocks can be evicted on demand).

---

## Macros

### `Z_ChangeTag`

```c
#define Z_ChangeTag(p, t) \
{ \
    if (((memblock_t *)((byte *)(p) - sizeof(memblock_t)))->id != 0x1d4a11) \
        I_Error("Z_CT at " __FILE__ ":%i", __LINE__); \
    Z_ChangeTag2(p, t); \
};
```

Wraps `Z_ChangeTag2` with an inline `ZONEID` check that reports the **call site** file and line number in the error message, rather than the file/line inside `z_zone.c`. This is the form all other engine modules should use. The check is duplicated (both here and inside `Z_ChangeTag2`) to give better diagnostics.

---

## Dependencies

| Header       | Reason |
|--------------|--------|
| `<stdio.h>`  | For `FILE *` parameter type in `Z_FileDumpHeap`. |
| `i_system.h` | (Used in `z_zone.c`) `I_ZoneBase` provides the raw memory arena; `I_Error` is called on fatal allocation failures and integrity violations. |
| `doomdef.h`  | (Used in `z_zone.c`) Provides the `byte` typedef. |
