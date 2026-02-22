# File Overview

**Path:** `sndserv/wadread.h`
**RCS:** `$Id: wadread.h,v 1.3 1997/01/30 19:54:23 b1 Exp $`

`wadread.h` is the public interface header for the sound server's minimal WAD reader (`wadread.c`). It exposes exactly two functions: one to open a WAD file and one to retrieve a sound-effect lump from it. Everything else in `wadread.c` (lump directory management, internal loading) is kept private.

The header comments acknowledge the redundancy with the full engine's `w_wad.h`: *"Welcome to Department of Redundancy Department."* and *"Note: makes up for a nice w_wad.h."* The sound server runs as a separate process and therefore needs its own standalone WAD reader rather than sharing the main engine's.

---

## Global Variables

None declared in this header.

---

## Functions (Declarations)

### `void openwad(char* wadname)`

**Purpose:** Open a WAD file by path, validate its IWAD header, and build an internal in-memory lump directory. Must be called exactly once before any call to `getsfx()`. Terminates the process via `derror()` if the file cannot be opened or has an invalid header.

**Parameters:**
- `wadname` - Null-terminated path string to the WAD file (e.g., `"/usr/local/games/doom/doom2.wad"`).

**Returns:** void.

**Precondition:** Must be called before `loadlump()` or `getsfx()`.

---

### `void* getsfx(char* sfxname, int* len)`

**Purpose:** Load and return the PCM sample data for a named sound effect. Internally prepends `"ds"` to the short name to form the WAD lump name (e.g., `"pistol"` becomes `"dspistol"`), loads it from the WAD, strips the 8-byte DMX sound header, pads the data to a multiple of `SAMPLECOUNT` with silence bytes (`0x80`), and returns a pointer directly to the usable PCM bytes.

**Parameters:**
- `sfxname` - Short sound-effect name, up to 6 characters. Must not include the `"ds"` prefix; the function adds it.
- `len` - Output parameter. Receives the padded PCM data length in bytes (not including the 8-byte DMX header, and rounded up to the nearest multiple of `SAMPLECOUNT`).

**Returns:**
- On success: a `void*` pointer to the first PCM sample byte in a heap-allocated buffer. The caller should cast this to `unsigned char*`.
- On failure (lump not found): `NULL`-equivalent (behavior undefined if the sound name does not exist in the WAD, as `loadlump()` returns NULL which is then passed to `realloc()`).

**Notes from header comment:**
- The returned pointer points to the *start of the data*, meaning it already skips the DMX header.
- Returns `0` if the sfx was not found.
- Sound names should be no longer than 6 characters (to leave room for the 2-character `"ds"` prefix within the 8-character WAD lump name limit).
- All data is rounded up in size to the nearest `MIXBUFFERSIZE` (actually `SAMPLECOUNT` in the implementation) and padded with `0x80` bytes.
- The length of usable data is returned via `len`.

---

## Data Structures

None defined in this header.

---

## Include Guard

```c
#ifndef __WADREAD_H__
#define __WADREAD_H__
// ...
#endif
```

---

## Dependencies

| Module | Usage |
|---|---|
| None | This header includes nothing. It is a pure interface declaration. |

Files that include this header:

| File | Why |
|---|---|
| `soundsrv.c` | Calls `openwad()` from `grabdata()` and `getsfx()` from the sound-loading loop. |
| `wadread.c` | Includes its own header to verify that the implementations match the declared signatures. |
