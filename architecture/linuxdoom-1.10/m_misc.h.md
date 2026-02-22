# File Overview

**File:** `linuxdoom-1.10/m_misc.h`
**Module prefix:** `M_`

This header is the public interface for `m_misc.c`. It exposes the five functions that form the miscellaneous utilities module: file I/O, configuration management, screenshot capture, and text rendering. Any translation unit that needs to read/write files generically, save or load the config, take a screenshot, or draw proportional text should include this header.

---

## Global Variables

None. This header declares no global variables. All state (`defaults[]`, `numdefaults`, `defaultfile`, `usemouse`, `usejoystick`) is internal to `m_misc.c` or owned by other modules.

---

## Functions

### `M_WriteFile`

```c
boolean M_WriteFile(char const* name, void* source, int length);
```

**Purpose:** Write `length` bytes of data from `source` to the file at path `name`. Creates the file if it does not exist; truncates it if it does.

**Parameters:**
- `name` - null-terminated filesystem path.
- `source` - pointer to the source data buffer.
- `length` - number of bytes to write.

**Returns:** `true` if all bytes were successfully written; `false` on any error. Failures are silent (no `I_Error`).

---

### `M_ReadFile`

```c
int M_ReadFile(char const* name, byte** buffer);
```

**Purpose:** Read the entire contents of the file at `name` into a newly allocated zone-memory buffer. The allocated memory has `PU_STATIC` priority and must be freed by the caller when no longer needed.

**Parameters:**
- `name` - null-terminated filesystem path.
- `buffer` - output parameter set to point at the newly allocated data buffer.

**Returns:** The file size in bytes.

**Error behaviour:** Calls `I_Error` (fatal abort) if the file cannot be opened, stat-ed, or fully read. Unlike `M_WriteFile`, read errors are always fatal.

---

### `M_ScreenShot`

```c
void M_ScreenShot(void);
```

**Purpose:** Capture the current rendered frame and save it as a PCX image file. Automatically selects the next available filename in the series `DOOM00.pcx`...`DOOM99.pcx`. Displays a `"screen shot"` confirmation message in the player's HUD. Calls `I_Error` if all 100 filename slots are occupied.

**Parameters:** None.

**Returns:** Nothing.

---

### `M_LoadDefaults`

```c
void M_LoadDefaults(void);
```

**Purpose:** Initialize all engine configuration variables to their compiled-in defaults, then apply any overrides stored in the config file (`~/.doomrc` on Linux, or the path given by `-config`). Must be called early in startup before any subsystem that depends on config-controlled variables.

**Parameters:** None.

**Returns:** Nothing.

---

### `M_SaveDefaults`

```c
void M_SaveDefaults(void);
```

**Purpose:** Write all current configuration variable values back to the config file. Typically called on clean engine shutdown to persist settings changed during a session (video detail, screen size, key bindings, etc.). Silent no-op if the file cannot be written.

**Parameters:** None.

**Returns:** Nothing.

---

### `M_DrawText`

```c
int M_DrawText(int x, int y, boolean direct, char* string);
```

**Purpose:** Render a text string to the screen using the HUD proportional font. `HU_Init()` must have been called before this function is used.

**Parameters:**
- `x` - left edge X pixel coordinate for the first character.
- `y` - top edge Y pixel coordinate.
- `direct` - `true` to write directly to video memory (`V_DrawPatchDirect`); `false` to write to the back buffer (`V_DrawPatch`).
- `string` - null-terminated ASCII string to render. Characters are forced to uppercase internally.

**Returns:** The X coordinate immediately following the last rendered character, enabling callers to concatenate additional rendering on the same baseline.

---

## Data Structures

None defined in this header. The `default_t` struct and `pcx_t` struct are defined privately in `m_misc.c`.

---

## Dependencies

| Header | Purpose |
|--------|---------|
| `doomtype.h` | Provides `boolean` and `byte` types used in the declarations |
