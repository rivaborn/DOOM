# File Overview

`p_saveg.h` is the public header for the save game I/O subsystem declared in `p_saveg.c`. It exposes the eight archive/unarchive functions and the shared `save_p` pointer to any module that needs to participate in saving or loading a game.

The primary consumer is `g_game.c`, which allocates the save buffer, sets `save_p` to point at it, calls the archive functions in sequence to populate the buffer, and then writes it to disk. On load, it reads the disk data into the buffer, resets `save_p`, and calls the unarchive functions in the same order.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `byte*` | `save_p` | The current position within the save game buffer. Declared `extern`; the actual variable lives in `p_saveg.c`. All archive and unarchive functions advance this pointer sequentially. |

## Functions (Declarations Only)

### `P_ArchivePlayers`
```c
void P_ArchivePlayers(void);
```
Serializes all active `player_t` structures to `save_p`. See `p_saveg.c` for implementation details.

### `P_UnArchivePlayers`
```c
void P_UnArchivePlayers(void);
```
Deserializes all active `player_t` structures from `save_p`.

### `P_ArchiveWorld`
```c
void P_ArchiveWorld(void);
```
Serializes the mutable parts of the map world (sector properties, line/side textures and offsets) to `save_p`.

### `P_UnArchiveWorld`
```c
void P_UnArchiveWorld(void);
```
Deserializes sector and sidedef state from `save_p`.

### `P_ArchiveThinkers`
```c
void P_ArchiveThinkers(void);
```
Serializes all `P_MobjThinker`-based thinkers (live map objects) to `save_p`.

### `P_UnArchiveThinkers`
```c
void P_UnArchiveThinkers(void);
```
Clears the current thinker list and restores mobjs from `save_p`.

### `P_ArchiveSpecials`
```c
void P_ArchiveSpecials(void);
```
Serializes all active sector-effect thinkers (ceilings, doors, floors, platforms, lights) to `save_p`.

### `P_UnArchiveSpecials`
```c
void P_UnArchiveSpecials(void);
```
Restores active sector-effect thinkers from `save_p`.

## Dependencies

| File | Reason |
|------|--------|
| (none explicit) | This header stands alone; the implementations in `p_saveg.c` pull in their own includes. Any consumer of this header must also include the headers defining `byte`. |
