# File Overview

`w_wad.h` is the public interface header for DOOM's WAD file system. It declares the three data structures that define the WAD format and the engine's lump directory, the three global variables that form the runtime lump database, and all the functions for querying and accessing lump data.

This header is included by virtually every module that needs to load game assets, making it one of the most widely-included headers in the codebase. Nearly all subsystems — rendering, sound, music, UI, levels, entities — use WAD lumps for their data.

---

## Global Variables

### `lumpcache`

```c
extern void** lumpcache;
```

**Type:** Pointer to an array of `void*` pointers, indexed by lump number.

**Purpose:** The lump data cache. `lumpcache[i]` is NULL if lump `i` is not currently loaded, or a pointer to the zone-allocated buffer containing lump `i`'s data if it is cached. The zone allocator automatically sets entries back to NULL when it purges the corresponding memory block.

---

### `lumpinfo`

```c
extern lumpinfo_t* lumpinfo;
```

**Type:** Pointer to a dynamically allocated array of `lumpinfo_t`.

**Purpose:** The global lump directory. One `lumpinfo_t` entry per lump, combining data from all loaded WAD files' directories. Indexed 0 to `numlumps - 1`.

---

### `numlumps`

```c
extern int numlumps;
```

**Type:** `int`.

**Purpose:** The total count of lumps across all loaded WAD files. The valid range for all lump indices is `[0, numlumps)`.

---

## Functions

### `W_InitMultipleFiles`

```c
void W_InitMultipleFiles(char** filenames);
```

**Purpose:** Primary WAD initialization function. Opens and processes a null-terminated array of WAD/lump file paths, building the global lump directory and allocating the lump cache.

**Parameters:**
- `filenames` - Null-terminated array of file path strings. Processed in order; later WADs override earlier ones during name searches (because `W_CheckNumForName` scans backward).

**Return value:** None.

**Key logic:** Calls `W_AddFile` for each entry, then allocates and zeroes `lumpcache`. Calls `I_Error` if no lumps were found.

---

### `W_Reload`

```c
void W_Reload(void);
```

**Purpose:** Flushes cached data for the reloadable WAD (the one whose filename was prefixed with `~`) and re-reads its directory. Allows live WAD replacement during development.

**Parameters:** None.

**Return value:** None.

**Key logic:** Only acts if a reloadable WAD was registered during `W_InitMultipleFiles`. Frees all cache entries for the reloadable lumps and updates their directory entries.

---

### `W_CheckNumForName`

```c
int W_CheckNumForName(char* name);
```

**Purpose:** Looks up a lump by name and returns its index, or -1 if not found. The search is case-insensitive and scans backward (so PWAD lumps override IWAD lumps).

**Parameters:**
- `name` - Lump name string (up to 8 characters).

**Return value:** Lump index (0-based), or -1 if not found. Callers must check for -1.

---

### `W_GetNumForName`

```c
int W_GetNumForName(char* name);
```

**Purpose:** Like `W_CheckNumForName` but calls `I_Error` if the lump is missing. Used when the lump is guaranteed to exist (e.g., core game resources).

**Parameters:**
- `name` - Lump name string.

**Return value:** Lump index. Never returns -1.

---

### `W_LumpLength`

```c
int W_LumpLength(int lump);
```

**Purpose:** Returns the uncompressed byte size of a lump.

**Parameters:**
- `lump` - Lump index.

**Return value:** Size of the lump in bytes.

---

### `W_ReadLump`

```c
void W_ReadLump(int lump, void* dest);
```

**Purpose:** Reads a lump's raw data from disk into a caller-supplied buffer. Low-level function; most code should prefer `W_CacheLumpNum` or `W_CacheLumpName`.

**Parameters:**
- `lump` - Lump index.
- `dest` - Destination buffer, must be at least `W_LumpLength(lump)` bytes.

**Return value:** None.

---

### `W_CacheLumpNum`

```c
void* W_CacheLumpNum(int lump, int tag);
```

**Purpose:** The primary lump access function. Returns a pointer to the lump's data in zone memory, loading from disk on first access (cache miss) or adjusting the zone tag on subsequent accesses (cache hit).

**Parameters:**
- `lump` - Lump index.
- `tag` - Zone memory tag controlling lump lifetime:
  - `PU_STATIC` — Lives until explicitly freed; never purged automatically.
  - `PU_LEVEL` — Lives until the end of the current level.
  - `PU_CACHE` — May be purged automatically when zone memory is needed.

**Return value:** Pointer to the lump's data. Valid until the zone block is freed or purged (which automatically nullifies `lumpcache[lump]`).

---

### `W_CacheLumpName`

```c
void* W_CacheLumpName(char* name, int tag);
```

**Purpose:** Convenience function that combines `W_GetNumForName` and `W_CacheLumpNum`. The most commonly used lump access function throughout the codebase.

**Parameters:**
- `name` - Lump name string.
- `tag` - Zone memory tag (same values as `W_CacheLumpNum`).

**Return value:** Pointer to the lump's data in zone memory.

**Usage example:**
```c
patch_t* mypatch = W_CacheLumpName("STBAR", PU_STATIC);
```

---

## Data Structures

### `wadinfo_t`

```c
typedef struct {
    char identification[4];   // "IWAD" or "PWAD"
    int  numlumps;            // Number of lumps in this WAD
    int  infotableofs;        // Byte offset of the lump directory
} wadinfo_t;
```

**Purpose:** The 12-byte on-disk header of a WAD file. The `identification` field distinguishes the main game data file (`"IWAD"`) from addon/patch files (`"PWAD"`). The directory (array of `filelump_t`) is located at `infotableofs` bytes from the beginning of the file.

---

### `filelump_t`

```c
typedef struct {
    int  filepos;   // Offset from file start to this lump's data
    int  size;      // Size of the lump data in bytes
    char name[8];   // Lump name (null-padded, NOT null-terminated if 8 chars long)
} filelump_t;
```

**Purpose:** One entry in the WAD file's on-disk lump directory. Each `filelump_t` is exactly 16 bytes. The directory is an array of `numlumps` of these records.

**Name format:** The 8-byte name is stored in uppercase. If the name is fewer than 8 characters, the remaining bytes are null. If exactly 8 characters, there is no null terminator. The engine handles this by using `strncpy` with length 8 and never relying on null termination.

---

### `lumpinfo_t`

```c
typedef struct {
    char name[8];    // Lump name (uppercase, null-padded)
    int  handle;     // POSIX file descriptor, or -1 for reloadable lumps
    int  position;   // Byte offset of lump data within its file
    int  size;       // Size of lump data in bytes
} lumpinfo_t;
```

**Purpose:** The engine's internal per-lump descriptor. Populated from `filelump_t` entries during WAD loading, with the addition of the file handle. The `lumpinfo` array is indexed by lump number and persists for the lifetime of the process.

**Handle field:** Normally holds the open POSIX file descriptor for the WAD file. For reloadable lumps (from a WAD loaded with a `~` prefix), this is -1, indicating that `W_ReadLump` must open and close the file on each access.

---

## Dependencies

| File | Reason |
|------|--------|
| `<stddef.h>` or compiler builtins | The `#pragma interface` / `#pragma implementation` directives (for GNU C++ ABI, used when compiled as C++) require no extra headers but affect compilation. |

This header is intentionally minimal and self-contained. It declares only the types and functions it defines, keeping dependencies out of the header itself. The implementations in `w_wad.c` pull in `z_zone.h`, `i_system.h`, and POSIX headers, but those are not exposed to users of this header.
