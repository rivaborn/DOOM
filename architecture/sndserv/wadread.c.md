# File Overview

**Path:** `sndserv/wadread.c`
**RCS:** `$Id: wadread.c,v 1.3 1997/01/30 19:54:23 b1 Exp $`

`wadread.c` is a minimal, self-contained WAD file reader written exclusively for the DOOM sound server (`sndserv`). It duplicates a small subset of the full WAD-management system from `linuxdoom-1.10/w_wad.c` — the id Software comments acknowledge this directly: *"Welcome to Department of Redundancy Department."*

Its sole job is to:
1. Open a WAD file and build an in-memory lump directory.
2. Load individual raw lumps by name.
3. Load, pad, and return sound-effect sample data by short name.

The sound server needs its own WAD reader because it runs as a completely separate process and cannot share the main DOOM process's memory or WAD cache.

---

## Global Variables

| Type | Name | Purpose |
|---|---|---|
| `int*` | `sfxlengths` | Pointer to an array of sound-effect lengths. Declared but never assigned or used within this file. Appears to be vestigial — the actual per-effect lengths are passed back through the `len` output parameter of `getsfx()`. |
| `lumpinfo_t*` | `lumpinfo` | Heap-allocated array of `lumpinfo_t` structs, one per lump in the open WAD. Built by `openwad()`. Used by `loadlump()` to locate lump data. |
| `int` | `numlumps` | Number of entries in the `lumpinfo[]` array; read from the WAD header by `openwad()`. |
| `void**` | `lumpcache` | Declared but never allocated or used in this file. Vestigial from the full `w_wad.c` caching design. |
| `static const char` | `rcsid[]` | RCS version string embedded in the binary for version identification. Not used at runtime. |

---

## Functions

### `static void derror(char* msg)`

**Purpose:** Fatal error handler specific to the WAD reader. Prints `"\nwadread error: <msg>\n"` to `stderr` and exits with status `-1`.

**Parameters:**
- `msg` - Human-readable error description.

**Returns:** Never returns.

---

### `unsigned long SwapLONG(unsigned long x)` (big-endian only)

**Purpose:** Byte-swaps a 32-bit unsigned integer from big-endian to little-endian (or vice versa). Only compiled when `__BIG_ENDIAN__` is defined.

**Parameters:**
- `x` - A 32-bit value to byte-swap.

**Returns:** Byte-swapped 32-bit value.

**Key Logic:** Manual shift-and-mask: `(x>>24) | ((x>>8)&0xff00) | ((x<<8)&0xff0000) | (x<<24)`.

---

### `unsigned short SwapSHORT(unsigned short x)` (big-endian only)

**Purpose:** Byte-swaps a 16-bit unsigned integer. Only compiled when `__BIG_ENDIAN__` is defined.

**Parameters:**
- `x` - A 16-bit value to byte-swap.

**Returns:** Byte-swapped 16-bit value: `(x>>8) | (x<<8)`.

---

### `void strupr(char* s)`

**Purpose:** Converts a null-terminated string to uppercase in place.

**Parameters:**
- `s` - The string to modify.

**Returns:** void.

**Note:** The implementation has a subtle bug: `*s++ = toupper(*s)` reads `*s` after incrementing `s` due to C's unspecified evaluation order for side effects. In practice on most platforms `*s` is read before the increment, making it equivalent to `*s = toupper(*s); s++`, but this is technically undefined behavior.

---

### `int filelength(int handle)`

**Purpose:** Returns the total byte size of an open file.

**Parameters:**
- `handle` - An open POSIX file descriptor.

**Returns:** The file size in bytes (`fileinfo.st_size`). Prints an error to `stderr` if `fstat()` fails but does not abort.

---

### `void openwad(char* wadname)`

**Purpose:** Opens a WAD file, validates its header, reads the lump directory into a heap-allocated `lumpinfo[]` array, and keeps the file descriptor open for subsequent lump reads.

**Parameters:**
- `wadname` - Path to the WAD file to open.

**Returns:** void. Calls `derror()` on failure.

**Key Logic:**

1. Opens the file with `open(wadname, O_RDONLY)`. Aborts if the open fails.
2. Reads the `wadinfo_t` header (12 bytes). Aborts if the 4-byte identification field is not `"IWAD"`.
3. Reads `numlumps` and `infotableofs` (byte-swapped via `LONG()` macro on big-endian platforms).
4. **Memory layout trick:** A single `malloc()` allocates enough space for the full `lumpinfo_t[]` array. The on-disk lump directory entries (`filelump_t`, which are smaller) are temporarily read into the *end* of that same allocation to avoid a second allocation:
   ```
   [ lumpinfo_t[0..n]  ... | filelump_t[0..n] ]
   ^                         ^
   lumpinfo                  filetable
   ```
   `filetable = (filelump_t*)((char*)lumpinfo + tablelength - tablefilelength)`
5. `lseek()`s to `infotableofs` and reads `tablefilelength` bytes of `filelump_t` entries into the end of the buffer.
6. Iterates over all lumps: copies the name, stores the WAD file descriptor as `handle`, and byte-swaps `filepos` and `size` into the lower (final) portion of the buffer, effectively converting in-place from `filelump_t` to `lumpinfo_t` layout as it goes.

---

### `void* loadlump(char* lumpname, int* size)`

**Purpose:** Finds a lump by name in the currently open WAD and reads its raw data into a heap-allocated buffer.

**Parameters:**
- `lumpname` - Up to 8-character lump name (case-insensitive comparison).
- `size` - Output parameter; receives the lump size in bytes if found.

**Returns:** Pointer to a heap-allocated buffer containing the raw lump data, or `NULL` if the lump is not found.

**Key Logic:**

- Linear scan of `lumpinfo[0..numlumps-1]` using `strncasecmp()` against the first 8 bytes of each entry's name.
- On a match: `malloc(lumpinfo[i].size)`, `lseek()` to `lumpinfo[i].filepos`, `read()` the full lump, set `*size`, return pointer.
- On no match: returns `NULL` (pointer set to `0`) without modifying `*size`.

---

### `void* getsfx(char* sfxname, int* len)`

**Purpose:** High-level sound-effect loader. Prepends `"ds"` to the short name, loads the WAD lump, strips the 8-byte DMX sound header, pads the raw PCM data up to a multiple of `SAMPLECOUNT`, and returns a pointer to the usable sample bytes.

**Parameters:**
- `sfxname` - Short sound name (up to 6 characters), e.g. `"pistol"`. The function prepends `"ds"` to form the WAD lump name `"dspistol"`.
- `len` - Output parameter; receives the padded sample length in bytes.

**Returns:** Pointer to the first usable sample byte within a heap-allocated buffer (i.e., 8 bytes past the start of the raw lump, skipping the DMX header). Returns `NULL`-equivalent if the lump was not found (since `loadlump()` returns NULL and this is passed to `realloc()`).

**Key Logic:**

1. Constructs `name = "ds" + sfxname` using `sprintf()`.
2. Calls `loadlump(name, &size)` to get the raw DMX sound lump.
3. **Padding:** `paddedsize = ((size - 8 + SAMPLECOUNT - 1) / SAMPLECOUNT) * SAMPLECOUNT`. This rounds the PCM data length (excluding the 8-byte header) up to the next multiple of `SAMPLECOUNT` (512). The purpose is to ensure that `mix()` never reads past the end of a sound's data during a mixing pass — the final partial mix-buffer's worth of samples will always have valid (silent) data to read.
4. `realloc()`s the buffer to `paddedsize + 8` and fills the padding bytes (`index size` through `paddedsize+7`) with `128` (the silence value for unsigned 8-bit PCM, where `128` represents 0).
5. Returns `paddedsfx + 8`, pointing past the DMX header directly at the PCM sample data.
6. Sets `*len = paddedsize` (the usable padded PCM length, not including the header).

**DMX Sound Format (8-byte header):**

| Offset | Size | Field |
|---|---|---|
| 0 | 2 bytes | Format identifier (always `3`) |
| 2 | 2 bytes | Sample rate in Hz |
| 4 | 4 bytes | Number of sample bytes |

The 8 header bytes are present in the allocated buffer but are never read by the mixer; only `paddedsfx + 8` onwards is used.

---

## Data Structures

### `wadinfo_t`

```c
typedef struct wadinfo_struct {
    char identification[4];  // "IWAD" for a main WAD file
    int  numlumps;            // Total number of lumps
    int  infotableofs;        // Byte offset of lump directory in file
} wadinfo_t;
```

Represents the 12-byte header at the start of every WAD file. Read once by `openwad()`.

### `filelump_t`

```c
typedef struct filelump_struct {
    int  filepos;  // Byte offset of this lump in the WAD file
    int  size;     // Size of the lump data in bytes
    char name[8];  // Lump name (up to 8 chars, not null-terminated)
} filelump_t;
```

On-disk representation of a lump directory entry (20 bytes on disk). Temporarily read into the tail of the `lumpinfo` allocation during `openwad()`, then converted to `lumpinfo_t` format in place.

### `lumpinfo_t`

```c
typedef struct lumpinfo_struct {
    int  handle;   // Open file descriptor of the WAD containing this lump
    int  filepos;  // Byte offset of lump data in the WAD file
    int  size;     // Size of the lump data in bytes
    char name[8];  // Lump name (up to 8 chars, not null-terminated)
} lumpinfo_t;
```

In-memory lump directory entry (24 bytes). Extends `filelump_t` with a `handle` field so that in a hypothetical multi-WAD setup each lump could reference its source file. In practice `sndserv` only ever opens one WAD.

---

## Endianness Macros

```c
// Little-endian (default, no-op):
#define LONG(x)  (x)
#define SHORT(x) (x)

// Big-endian (compiled when __BIG_ENDIAN__ is defined):
#define LONG(x)  ((long)SwapLONG((unsigned long)(x)))
#define SHORT(x) ((short)SwapSHORT((unsigned short)(x)))
```

WAD files are always stored in little-endian byte order. On big-endian platforms (e.g., SGI IRIX MIPS) these macros call `SwapLONG()`/`SwapSHORT()` to correct the byte order when reading the WAD header and lump directory. `SHORT` is defined but not called in this file (no 16-bit WAD fields are read here); it is provided for completeness.

The `strcmpi` macro is defined as an alias for the POSIX `strcasecmp`:

```c
#define strcmpi strcasecmp
```

---

## Dependencies

| Module | Usage |
|---|---|
| `soundsrv.h` | Provides the `SAMPLECOUNT` constant used in `getsfx()` to compute the padded buffer size. |
| `wadread.h` | This file's own interface header; included to ensure the function signatures match their declarations. |
| `<malloc.h>` | `malloc()`, `realloc()`. |
| `<fcntl.h>` | `open()`, `O_RDONLY`. |
| `<sys/stat.h>` | `fstat()`, `struct stat`. |
| `<stdio.h>` | `fprintf()`, `sprintf()`. |
| `<string.h>` | `strncpy()`. |
| `<stdlib.h>` | `exit()`. |
| `<ctype.h>` | `toupper()` used in `strupr()`. |
| `<unistd.h>` | `read()`, `lseek()`, `close()`. |
