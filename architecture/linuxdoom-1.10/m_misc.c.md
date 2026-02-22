# File Overview

**File:** `linuxdoom-1.10/m_misc.c`
**Module prefix:** `M_`

`m_misc.c` is a catch-all utility module that provides three distinct subsystems used across the entire DOOM engine:

1. **File I/O helpers** - generic binary file read and write routines used by save-games, screenshots, and other subsystems.
2. **Configuration file (defaults) system** - reads and writes a human-readable key/value config file (`~/.doomrc` on Linux, `default.cfg` on DOS) that persists player preferences across sessions. The defaults table acts as the single authoritative mapping between config-file keys and the live in-memory variables they control.
3. **PCX screenshot capture** - grabs the current framebuffer, run-length encodes it into a PCX file, and writes it to disk as `DOOM00.pcx`...`DOOM99.pcx`.
4. **Text rendering helper** - `M_DrawText` draws a string to the screen using the HUD proportional font.

---

## Global Variables

| Type | Name | Purpose |
|------|------|---------|
| `int` | `usemouse` | Whether mouse input is enabled (0 = off, 1 = on). Persisted in config. |
| `int` | `usejoystick` | Whether joystick input is enabled (0 = off, 1 = on). Persisted in config. |
| `char*` | `mousetype` | (Linux only, `#ifdef LINUX`) Serial mouse protocol string, e.g. `"microsoft"`. Persisted in config. |
| `char*` | `mousedev` | (Linux only, `#ifdef LINUX`) Serial mouse device path, e.g. `"/dev/ttyS0"`. Persisted in config. |
| `int` | `numdefaults` | Number of entries in the `defaults[]` table. Computed once in `M_LoadDefaults`. |
| `char*` | `defaultfile` | Resolved path to the config file that will be read/written. Set to `basedefault` or to the `-config` command-line argument. |
| `default_t` | `defaults[]` | The master defaults table (see Data Structures below). Contains one entry per config-file key. |

### Externally owned variables referenced by `defaults[]`

The defaults table holds pointers to variables owned by other modules. Every variable listed below is read from or written to the config file:

| Variable | Owner module | Config key |
|----------|-------------|------------|
| `mouseSensitivity` | `g_game.c` | `mouse_sensitivity` |
| `snd_SfxVolume` | `s_sound.c` | `sfx_volume` |
| `snd_MusicVolume` | `s_sound.c` | `music_volume` |
| `showMessages` | `g_game.c` | `show_messages` |
| `key_right/left/up/down` | `g_game.c` | `key_right` etc. |
| `key_strafeleft/right` | `g_game.c` | `key_strafeleft/right` |
| `key_fire/use/strafe/speed` | `g_game.c` | `key_fire` etc. |
| `mousebfire/strafe/forward` | `g_game.c` | `mouseb_fire` etc. |
| `joybfire/strafe/use/speed` | `g_game.c` | `joyb_fire` etc. |
| `screenblocks` | `g_game.c` | `screenblocks` |
| `detailLevel` | `r_draw.c` | `detaillevel` |
| `numChannels` | `s_sound.c` | `snd_channels` |
| `usegamma` | `v_video.c` | `usegamma` |
| `chat_macros[0..9]` | `hu_stuff.c` | `chatmacro0`..`chatmacro9` |
| `sndserver_filename` | `i_sound.c` | `sndserver` (SNDSERV only) |
| `mb_used` | `i_sound.c` | `mb_used` (SNDSERV only) |
| `mousetype` | local | `mousetype` (Linux only) |
| `mousedev` | local | `mousedev` (Linux only) |

---

## Functions

### `M_DrawText`

```c
int M_DrawText(int x, int y, boolean direct, char* string)
```

**Purpose:** Renders an ASCII string to the screen using the HUD proportional font (`hu_font[]`). Used for in-game text overlays.

**Parameters:**
- `x` - starting X pixel coordinate.
- `y` - Y pixel coordinate (top of glyphs).
- `direct` - if `true`, calls `V_DrawPatchDirect` (writes straight to video memory); if `false`, calls `V_DrawPatch` (writes to the back buffer).
- `string` - null-terminated ASCII string to render.

**Returns:** The X coordinate immediately after the last character drawn, allowing the caller to continue rendering on the same line.

**Key logic:**
- Iterates character by character, converting each to uppercase.
- Subtracts `HU_FONTSTART` to map the character to a font patch index.
- Characters outside the font range advance X by 4 pixels (space substitute).
- Stops early if the next glyph would extend past `SCREENWIDTH`.
- Requires `HU_Init()` to have been called first so `hu_font[]` is populated.

---

### `M_WriteFile`

```c
boolean M_WriteFile(char const* name, void* source, int length)
```

**Purpose:** Writes `length` bytes from `source` to the file at `name`, creating or truncating the file. Used by screenshot capture and save-game writing.

**Parameters:**
- `name` - filesystem path for the output file.
- `source` - pointer to the data buffer to write.
- `length` - number of bytes to write.

**Returns:** `true` on success; `false` if the file could not be opened or fewer than `length` bytes were written.

**Key logic:**
- Opens with `O_WRONLY | O_CREAT | O_TRUNC | O_BINARY` (O_BINARY is defined to 0 on non-DOS systems).
- Silent failure: returns `false` without calling `I_Error`, so the caller decides whether to report the error.

---

### `M_ReadFile`

```c
int M_ReadFile(char const* name, byte** buffer)
```

**Purpose:** Reads an entire file into a zone-memory buffer allocated with `PU_STATIC`. Used for WAD loading helpers and other bulk-read operations.

**Parameters:**
- `name` - path of the file to read.
- `buffer` - output parameter; set to point at the newly allocated buffer containing the file data.

**Returns:** The length of the file in bytes.

**Key logic:**
- Uses `fstat` to determine the file size before allocating.
- Allocates with `Z_Malloc(length, PU_STATIC, NULL)` - caller is responsible for freeing.
- Calls `I_Error` (fatal) on any failure (file not found, stat error, short read). Unlike `M_WriteFile`, read failures are always fatal.

---

### `M_SaveDefaults`

```c
void M_SaveDefaults(void)
```

**Purpose:** Writes all current in-memory default values out to the config file. Called on clean exit so the player's settings are remembered.

**Parameters:** None.

**Returns:** Nothing. Silent no-op if the config file cannot be opened for writing (e.g., read-only filesystem).

**Key logic:**
- Iterates all `numdefaults` entries in `defaults[]`.
- **Integer vs. string detection:** if `defaults[i].defaultvalue` is in the range `(-0xfff, 0xfff)`, the entry is treated as an integer and written as `name\t\tvalue\n`. Otherwise, `defaults[i].location` is cast to `char**` and the string value is written quoted as `name\t\t"value"\n`.
- This numeric-range hack exists because string entries store a pointer cast to `int` in `defaultvalue` (a large number outside the integer range), which serves as a sentinel.

---

### `M_LoadDefaults`

```c
void M_LoadDefaults(void)
```

**Purpose:** Initializes all config variables to their compiled-in defaults, then reads the config file and overrides them with the saved values. Must be called early in engine startup, before any subsystem that reads config variables.

**Parameters:** None.

**Returns:** Nothing.

**Key logic:**

1. **Compute table size:** `numdefaults = sizeof(defaults)/sizeof(defaults[0])`.
2. **Apply compiled-in defaults:** for each entry, writes `defaultvalue` directly into `*location`. For string entries this stores a pointer to a string literal.
3. **Resolve config file path:** checks for `-config <path>` command-line argument; falls back to `basedefault` (set by `d_main.c` to `~/.doomrc` on Linux).
4. **Parse config file:** uses `fscanf(f, "%79s %[^\n]\n", def, strparm)` to read one key/value pair per line.
   - If `strparm` starts with `"`, the value is a string: allocates a new heap buffer with `malloc`, strips the surrounding quotes, and stores the pointer cast to `int` into `*location`.
   - If `strparm` starts with `0x`, parses as hexadecimal.
   - Otherwise parses as decimal integer.
5. Silently ignores unrecognised keys and a missing config file.

---

### `WritePCXfile` (static/internal)

```c
void WritePCXfile(char* filename, byte* data, int width, int height, byte* palette)
```

**Purpose:** Encodes raw 8-bit indexed pixel data into PCX format and writes it to disk. Called exclusively by `M_ScreenShot`.

**Parameters:**
- `filename` - output filename.
- `data` - raw pixel data, `width * height` bytes, row-major, 8-bit palette indices.
- `width`, `height` - image dimensions in pixels.
- `palette` - 768-byte RGB palette (256 entries x 3 bytes).

**Returns:** Nothing.

**Key logic:**
- Allocates a temporary buffer of `width*height*2 + 1000` bytes from zone memory to hold the worst-case encoded output.
- Fills a `pcx_t` header: manufacturer `0x0a`, version 5 (256-colour), encoding 1 (RLE), 8 bpp, 1 colour plane.
- **RLE encoding:** any byte with bits `0xC0` both set is escaped as `0xC1, value` (a run of 1). Bytes without those bits set are stored as-is. This is the minimal PCX RLE encoding - it never produces multi-byte runs.
- Appends the 768-byte palette preceded by the `0x0C` palette marker byte.
- Uses `M_WriteFile` to write the result; frees the zone buffer immediately after.

---

### `M_ScreenShot`

```c
void M_ScreenShot(void)
```

**Purpose:** Captures the current screen, encodes it as PCX, writes it to a sequentially numbered file, and displays a "screen shot" message to the player.

**Parameters:** None.

**Returns:** Nothing.

**Key logic:**
- Uses `screens[2]` (a third off-screen buffer) as a scratch area.
- Calls `I_ReadScreen(linear)` to copy the current video output into the scratch buffer, handling any platform-specific planar-to-linear conversion.
- Iterates filenames `DOOM00.pcx`...`DOOM99.pcx`, using `access(name, 0)` to find the first that does not exist.
- Calls `I_Error` if all 100 slots are taken.
- Fetches the current palette from the WAD lump `PLAYPAL` with `W_CacheLumpName("PLAYPAL", PU_CACHE)`.
- Sets `players[consoleplayer].message` to display the confirmation message on the HUD.

---

## Data Structures

### `default_t`

```c
typedef struct
{
    char*   name;           // Config file key string, e.g. "mouse_sensitivity"
    int*    location;       // Pointer to the live in-memory variable
    int     defaultvalue;   // Compiled-in default; also used as string sentinel
    int     scantranslate;  // PC scan-code translation flag (unused in Linux port)
    int     untranslated;   // Stores original untranslated scan code (unused in Linux port)
} default_t;
```

This structure is the core of the defaults system. An array of `default_t` named `defaults[]` enumerates every configurable engine parameter. The `location` pointer creates a direct link between the config file key and the live variable - reading or writing `*location` immediately affects engine behaviour.

**String vs. integer duality:** For string values (e.g., `sndserver`, `chatmacro0`), `location` is cast to `int*` but actually points to a `char*` variable. `defaultvalue` stores the original pointer to the string literal cast to `int`. The range check `defaultvalue > -0xfff && defaultvalue < 0xfff` in `M_SaveDefaults` distinguishes integer entries (small values) from string entries (large pointer values).

---

### `pcx_t`

```c
typedef struct
{
    char            manufacturer;       // Must be 0x0a (ZSoft PCX magic)
    char            version;            // 5 = 256-colour PCX
    char            encoding;           // 1 = RLE encoded
    char            bits_per_pixel;     // 8
    unsigned short  xmin, ymin;         // Image origin (always 0,0)
    unsigned short  xmax, ymax;         // Image extents (width-1, height-1)
    unsigned short  hres, vres;         // Horizontal/vertical DPI (set to width/height)
    unsigned char   palette[48];        // Unused 16-colour EGA palette (zeroed)
    char            reserved;           // Must be 0
    char            color_planes;       // 1 (chunky/planar plane count)
    unsigned short  bytes_per_line;     // Bytes per scan line (= width)
    unsigned short  palette_type;       // 2 = colour image
    char            filler[58];         // Padding to 128-byte header
    unsigned char   data;               // First byte of image data (unbounded)
} pcx_t;
```

Standard ZSoft PCX file header. DOOM uses 8-bit 256-colour PCX (version 5) with a trailing 768-byte palette appended after the image data (preceded by the `0x0C` sentinel byte).

---

## Dependencies

| Module | Header | What is used |
|--------|--------|-------------|
| Zone allocator | `z_zone.h` | `Z_Malloc`, `Z_Free` for screenshot buffer and file read buffer |
| Byte swap | `m_swap.h` | `SHORT()` macro for PCX header fields |
| Argument parsing | `m_argv.h` | `M_CheckParm`, `myargc`, `myargv` for `-config` argument |
| WAD file system | `w_wad.h` | `W_CacheLumpName` to fetch `PLAYPAL` palette |
| System layer | `i_system.h` | `I_Error` for fatal errors |
| Video I/O | `i_video.h` | `I_ReadScreen` to capture framebuffer |
| Video draw | `v_video.h` | `V_DrawPatch`, `V_DrawPatchDirect`, `screens[]` array |
| HUD | `hu_stuff.h` | `hu_font[]` patch array, `HU_FONTSIZE`, `HU_FONTSTART` |
| Game state | `doomstat.h` | `players[]`, `consoleplayer`, `basedefault`, `usegamma` |
| Engine defs | `doomdef.h` | `SCREENWIDTH`, `SCREENHEIGHT`, `boolean`, `byte`, `patch_t` |
| String data | `dstrings.h` | `HUSTR_CHATMACROn` default chat macro strings |
| Sound state | _(extern)_ | `snd_SfxVolume`, `snd_MusicVolume`, `numChannels` |
