# File Overview

`w_wad.c` implements the WAD (Where's All the Data?) file system, which is the foundation of all asset loading in DOOM. Every piece of game content — level geometry, sprites, sounds, music, textures, palettes, UI graphics, and strings — is stored in WAD files. This module handles opening WAD files, parsing their directory, and providing cached access to individual data lumps.

## The WAD Format

A WAD file has three sections:

1. **Header** (12 bytes): Contains a 4-byte identification string (`"IWAD"` for the main game data file, `"PWAD"` for patch/addon files), the number of lumps, and the file offset of the directory.

2. **Raw lump data**: Concatenated binary blobs of arbitrary size, in no particular format (each lump type has its own interpretation). The lumps are stored contiguously but do not need to be in any particular order relative to the directory.

3. **Directory**: An array of 16-byte `filelump_t` records, one per lump. Each record specifies the file offset, byte size, and 8-character name of a lump. The directory can be anywhere in the file (its offset is given in the header).

### IWAD vs PWAD

- **IWAD** (Internal WAD): The main game data file (`doom.wad`, `doom2.wad`, etc.). Contains all the base game lumps.
- **PWAD** (Patch WAD): An addon or level pack that overrides or supplements the IWAD. Multiple PWADs can be loaded. When searching for a lump by name, the engine scans the lump list **backwards** (from last-loaded to first), so PWADs automatically override IWADs.

### Lump Naming

Lump names are up to 8 characters, stored as null-padded (not null-terminated) ASCII in the WAD directory. Names are case-insensitive; the engine converts all names to uppercase for comparison. Some lumps use special naming conventions: map lumps are grouped between marker lumps (e.g., `E1M1` followed by `THINGS`, `LINEDEFS`, etc.), and sprite frames use a 4+2+2 naming scheme (`TROOA1`, `TROOA2A8`, etc.).

### Lump Cache

The module maintains a `lumpcache` array parallel to `lumpinfo`. Each entry is either NULL (lump not cached) or a pointer to zone-allocated memory holding the lump's data. The zone tag on cached lumps controls their lifetime: static lumps (`PU_STATIC`) persist indefinitely, cache lumps (`PU_CACHE`) can be freed automatically when memory is needed.

---

## Global Variables

### `lumpinfo`

```c
lumpinfo_t* lumpinfo;
```

**Type:** Pointer to a dynamically allocated array of `lumpinfo_t` structures.

**Purpose:** The engine's internal lump directory. One entry per lump across all loaded WAD files. Initialized empty and grown by `realloc` as each WAD file is added. Indexed from 0 to `numlumps - 1`.

---

### `numlumps`

```c
int numlumps;
```

**Type:** `int`.

**Purpose:** The total number of lumps currently loaded from all WAD files combined. Used as the upper bound for all lump index operations.

---

### `lumpcache`

```c
void** lumpcache;
```

**Type:** Pointer to an array of `void*` pointers.

**Purpose:** The lump cache table. `lumpcache[i]` is either NULL (lump `i` is not currently in memory) or a pointer to the zone-allocated buffer containing lump `i`'s data. Allocated in `W_InitMultipleFiles` as a zero-filled array of `numlumps` pointers.

The cache uses a two-pointer trick with the zone allocator: when `Z_Malloc` is called with a non-NULL `user` pointer (here `&lumpcache[lump]`), the zone allocator stores the user pointer and clears it to NULL if the block is purged. This means `lumpcache[i]` automatically becomes NULL when the zone reclaims that memory, allowing the cache check `if (!lumpcache[lump])` to correctly detect a cache miss.

---

### `reloadlump` (file-static)

```c
int reloadlump;
```

**Type:** `int`.

**Purpose:** The starting lump index of the "reloadable" WAD file. When a WAD filename is prefixed with `~`, that WAD's lumps are marked as reloadable — the file handle is closed after parsing the directory, and the file is reopened fresh on each read. This allows in-place replacement of WAD files during development without restarting the engine.

---

### `reloadname` (file-static)

```c
char* reloadname;
```

**Type:** `char*`.

**Purpose:** The filename of the reloadable WAD, or NULL if no reloadable WAD has been registered. When non-NULL, `W_ReadLump` opens this file fresh for each lump read in the reloadable range.

---

### `info[2500][10]` and `profilecount` (file-static)

```c
int info[2500][10];
int profilecount;
```

**Purpose:** Profiling data used by the `W_Profile` debug function. `info[i][j]` records whether lump `i` was static ('S') or purgeable ('P') at profiling snapshot `j`. Not part of the normal runtime path.

---

## Functions

### `strupr` (internal helper)

```c
void strupr(char* s);
```

**Purpose:** Converts a null-terminated string to uppercase in-place. Used to normalize lump names for case-insensitive comparison.

**Parameters:**
- `s` - String to convert.

**Return value:** None.

---

### `filelength` (internal helper)

```c
int filelength(int handle);
```

**Purpose:** Returns the size in bytes of an open file identified by a file descriptor.

**Parameters:**
- `handle` - An open POSIX file descriptor.

**Return value:** The file's size in bytes, obtained via `fstat`.

**Key logic:** Uses `fstat(handle, &fileinfo)` to get the file's `st_size`. Calls `I_Error` if `fstat` fails.

---

### `ExtractFileBase` (internal helper)

```c
void ExtractFileBase(char* path, char* dest);
```

**Purpose:** Extracts the base filename (no directory, no extension) from a full path string, converted to uppercase, and stores it in an 8-byte buffer. Used when loading a non-WAD file as a single lump — the lump name is derived from the filename.

**Parameters:**
- `path` - Full file path string (e.g., `/usr/share/doom/mymap.lmp`).
- `dest` - Output buffer for the lump name. Must be at least 8 bytes; will be null-padded to exactly 8 bytes.

**Return value:** None.

**Key logic:** Scans backward from the end of `path` to find the last directory separator (`\` or `/`), then copies forward up to 8 uppercase characters stopping at the first `.`. Calls `I_Error` if the base name is longer than 8 characters.

---

### `W_AddFile` (internal, not declared in header)

```c
void W_AddFile(char* filename);
```

**Purpose:** Opens a single file (WAD or raw lump) and appends its lump entries to the global `lumpinfo` array.

**Parameters:**
- `filename` - Path to the WAD or data file. If the filename starts with `~`, it is treated as a reloadable WAD.

**Return value:** None.

**Key logic:**

1. **Reloadable WAD detection**: If `filename[0] == '~'`, increments `filename` to skip the `~`, sets `reloadname` and `reloadlump`.

2. **File type detection**: If the filename does not end in `".wad"` (case-insensitive), the file is treated as a single-lump file. Otherwise it is parsed as a WAD.

3. **Single-lump path**: Creates a synthetic `filelump_t` with position 0, size = full file length, and name = `ExtractFileBase(filename)`. Increments `numlumps` by 1.

4. **WAD path**: Reads the 12-byte header, validates `"IWAD"` or `"PWAD"` identification, reads the `header.numlumps * 16` byte directory from `header.infotableofs`, and increments `numlumps` by `header.numlumps`.

5. **Lump table growth**: Calls `realloc(lumpinfo, numlumps * sizeof(lumpinfo_t))` to grow the lump table. Fills in the new entries with file position, size, handle, and 8-byte name from the directory.

6. **Reloadable handle management**: For reloadable WADs, `storehandle` is set to -1 (the file is closed after reading the directory). For normal WADs, `storehandle` is the open file descriptor, which is kept open for the lifetime of the process.

---

### `W_Reload`

```c
void W_Reload(void);
```

**Purpose:** Re-reads the directory of the reloadable WAD file and flushes any cached lumps from it. Allows the reloadable WAD to be replaced on disk and the new content to take effect without restarting the engine.

**Parameters:** None.

**Return value:** None.

**Key logic:** Only does anything if `reloadname` is non-NULL. Opens the reload file, reads its directory, then iterates over lumps from `reloadlump` onward, freeing any cached data with `Z_Free(lumpcache[i])` and updating each lump's `position` and `size` from the new directory. Closes the file when done.

---

### `W_InitMultipleFiles`

```c
void W_InitMultipleFiles(char** filenames);
```

**Purpose:** The main WAD initialization entry point. Loads a null-terminated array of WAD/lump filenames into the engine's lump system, sets up the lump cache, and readies the system for lump queries.

**Parameters:**
- `filenames` - A null-terminated array of file path strings. Processed in order; later files can override earlier ones.

**Return value:** None.

**Key logic:**

1. Initializes `numlumps = 0` and `lumpinfo = malloc(1)` (minimal allocation that can be `realloc`'d).
2. Calls `W_AddFile` for each filename in the array.
3. Verifies at least one lump was found, or calls `I_Error`.
4. Allocates and zeroes the `lumpcache` array: `lumpcache = malloc(numlumps * sizeof(void*))`.

---

### `W_InitFile` (not in header)

```c
void W_InitFile(char* filename);
```

**Purpose:** Convenience wrapper that initializes the WAD system from a single file. Constructs a two-element array `{filename, NULL}` and passes it to `W_InitMultipleFiles`.

---

### `W_NumLumps` (not in header)

```c
int W_NumLumps(void);
```

**Purpose:** Returns the total number of loaded lumps.

**Return value:** `numlumps`.

---

### `W_CheckNumForName`

```c
int W_CheckNumForName(char* name);
```

**Purpose:** Searches for a lump by name and returns its index, or -1 if not found. Case-insensitive.

**Parameters:**
- `name` - The lump name to search for (up to 8 characters; longer names are silently truncated to 8).

**Return value:** The lump index (0-based), or -1 if the name is not found.

**Key logic:**

1. Copies the name into a 9-byte union `name8` (to add a null terminator), converts to uppercase.
2. Reinterprets the 8-byte name as two 32-bit integers `v1` and `v2` for fast comparison.
3. **Scans the lump array backwards** (`lump_p = lumpinfo + numlumps`, then `while (lump_p-- != lumpinfo)`). This backward scan ensures that later-loaded WADs (PWADs) take precedence over earlier ones.
4. Compares the two 32-bit halves of each lump's name with `v1` and `v2` directly, treating the name as two consecutive `int` values. This integer-comparison trick is about 8x faster than `strncmp` on the architectures of the era.
5. Returns the matched index or -1.

---

### `W_GetNumForName`

```c
int W_GetNumForName(char* name);
```

**Purpose:** Like `W_CheckNumForName` but calls `I_Error` if the lump is not found. Used when the lump is known to be required.

**Parameters:**
- `name` - The lump name to find.

**Return value:** The lump index (never returns -1).

---

### `W_LumpLength`

```c
int W_LumpLength(int lump);
```

**Purpose:** Returns the size in bytes of the specified lump's data.

**Parameters:**
- `lump` - Lump index (0 to `numlumps - 1`).

**Return value:** The size in bytes of the lump's raw data.

**Key logic:** Validates `lump < numlumps`, then returns `lumpinfo[lump].size`.

---

### `W_ReadLump`

```c
void W_ReadLump(int lump, void* dest);
```

**Purpose:** Reads the raw binary data of a lump from disk into a caller-provided buffer.

**Parameters:**
- `lump` - Lump index.
- `dest` - Destination buffer. Must be at least `W_LumpLength(lump)` bytes.

**Return value:** None.

**Key logic:**

1. Validates `lump < numlumps`.
2. Checks `lumpinfo[lump].handle`. If -1, this is a reloadable lump: opens `reloadname` fresh, reads, then closes. Otherwise uses the stored handle.
3. Seeks to `lumpinfo[lump].position` with `lseek`.
4. Reads `lumpinfo[lump].size` bytes with `read`. Calls `I_Error` if the read is short.

---

### `W_CacheLumpNum`

```c
void* W_CacheLumpNum(int lump, int tag);
```

**Purpose:** The primary lump access function. Returns a pointer to the lump's data in memory, loading it from disk if necessary and caching it in the zone allocator with the specified tag.

**Parameters:**
- `lump` - Lump index.
- `tag` - Zone memory tag (e.g., `PU_STATIC`, `PU_CACHE`, `PU_LEVEL`). Controls how long the cached data survives.

**Return value:** Pointer to the lump's data in zone memory. The data persists as long as the zone block is not freed or purged.

**Key logic:**

- **Cache miss** (`lumpcache[lump] == NULL`): Calls `Z_Malloc(W_LumpLength(lump), tag, &lumpcache[lump])` to allocate zone memory, then `W_ReadLump(lump, lumpcache[lump])` to fill it. Passing `&lumpcache[lump]` as the `user` pointer to `Z_Malloc` registers it with the zone: if the block is later purged, `lumpcache[lump]` is set back to NULL automatically.
- **Cache hit** (`lumpcache[lump] != NULL`): Calls `Z_ChangeTag(lumpcache[lump], tag)` to potentially adjust the purge level of the existing cached block (e.g., to promote it from `PU_CACHE` to `PU_STATIC`).

---

### `W_CacheLumpName`

```c
void* W_CacheLumpName(char* name, int tag);
```

**Purpose:** Convenience wrapper combining `W_GetNumForName` and `W_CacheLumpNum`. The most common way to load a named lump.

**Parameters:**
- `name` - The lump name.
- `tag` - Zone memory tag.

**Return value:** Pointer to the lump's data in zone memory.

---

### `W_Profile` (debug, not in header)

```c
void W_Profile(void);
```

**Purpose:** Debug profiling function. Takes a snapshot of which lumps are cached and at what tag level, accumulates snapshots over multiple calls, and writes a formatted report to `waddump.txt`.

**Key logic:** Iterates all lumps; for each cached lump checks if the zone block tag is below `PU_PURGELEVEL` (marking it 'S' for static) or at/above (marking 'P' for purgeable). Writes all accumulated snapshots to file showing cache state over time.

---

## Data Structures

### `wadinfo_t`

```c
typedef struct {
    char identification[4];  // "IWAD" or "PWAD"
    int  numlumps;            // Total lump count
    int  infotableofs;        // File offset of the directory
} wadinfo_t;
```

**Purpose:** The 12-byte header at the very beginning of every WAD file. Read once per file during `W_AddFile`.

---

### `filelump_t`

```c
typedef struct {
    int  filepos;    // File offset of this lump's data
    int  size;       // Size of this lump in bytes
    char name[8];    // Lump name (null-padded, not null-terminated)
} filelump_t;
```

**Purpose:** One 16-byte entry in the WAD's on-disk directory. The directory is an array of these. Read from disk during WAD loading; not stored in memory after `W_AddFile` returns (uses `alloca` for temporary stack allocation).

---

### `lumpinfo_t`

```c
typedef struct {
    char name[8];    // Lump name (copied from filelump_t, null-padded)
    int  handle;     // Open file descriptor, or -1 for reloadable lumps
    int  position;   // File offset within the WAD file
    int  size;       // Lump size in bytes
} lumpinfo_t;
```

**Purpose:** The engine's internal lump descriptor, one per lump in the global `lumpinfo[]` array. Derived from `filelump_t` during loading with the addition of the file handle. Persists for the lifetime of the engine.

---

## Dependencies

| File | Reason |
|------|--------|
| `doomtype.h` | Provides `byte` and other basic types. |
| `m_swap.h` | `LONG()` macro for converting 32-bit integers from little-endian WAD format to host byte order. |
| `i_system.h` | `I_Error` for fatal error reporting. |
| `z_zone.h` | `Z_Malloc`, `Z_Free`, `Z_ChangeTag` for zone-allocated lump cache management. `memblock_t` used in `W_Profile`. |
| `w_wad.h` | Self-header providing the public interface and type declarations for `wadinfo_t`, `filelump_t`, `lumpinfo_t`. |
| POSIX headers | `<sys/types.h>`, `<sys/stat.h>`, `<fcntl.h>`, `<unistd.h>` for `open`, `read`, `lseek`, `close`, `fstat`. |
| `<ctype.h>` | `toupper` used in `strupr` and `ExtractFileBase`. |
| `<alloca.h>` | `alloca` for stack-allocating the temporary `filelump_t` array during WAD loading. |
