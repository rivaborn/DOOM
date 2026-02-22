# File Overview

`st_lib.c` is the status bar widget library for the DOOM engine. It provides a set of reusable UI widget primitives that the main status bar code (`st_stuff.c`) uses to display numeric values, percentage readouts, multi-state icons, and binary (on/off) icons on the 32-pixel-tall status bar at the bottom of the screen.

The design is data-driven and uses pointer indirection: each widget holds a pointer to the actual game data value it should display (e.g., `&plyr->health`) rather than a cached copy. On each frame, the widget checks whether its tracked value has changed and only redraws if necessary, optimizing screen updates. A `boolean* on` pointer controls whether the widget is currently visible at all (e.g., the frags counter is hidden in single-player mode).

All drawing is performed into screen buffer `FG` (screen 0, the display framebuffer), using background data from screen `BG` (screen 4, a cached copy of the status bar background) to erase the old value before drawing the new one.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `patch_t*` | `sttminus` | Loaded patch for the minus sign graphic (`STTMINUS` lump). Used to render negative frags counts. |

---

## Functions

### `STlib_init`

```c
void STlib_init(void)
```

**Purpose:** Initializes the widget library by loading the minus-sign graphic patch from the WAD.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** Loads the `"STTMINUS"` WAD lump into `sttminus` as a `PU_STATIC` allocation. This patch is used by `STlib_drawNum` to display negative numbers.

---

### `STlib_initNum`

```c
void STlib_initNum(st_number_t* n, int x, int y, patch_t** pl,
                   int* num, boolean* on, int width)
```

**Purpose:** Initializes a number widget (`st_number_t`) with its position, digit patches, data pointer, and display width.

**Parameters:**
- `n` — Pointer to the widget structure to initialize.
- `x` — X coordinate of the right edge of the number (numbers are right-justified).
- `y` — Y coordinate of the number.
- `pl` — Array of 10 patches for digit glyphs 0-9.
- `num` — Pointer to the integer value to display (e.g., `&plyr->health`).
- `on` — Pointer to a boolean controlling widget visibility.
- `width` — Maximum number of digits to display.

**Returns:** Nothing.

**Key Logic:** Copies all parameters directly into the widget structure. Sets `oldnum = 0` so the first update always redraws.

---

### `STlib_drawNum`

```c
void STlib_drawNum(st_number_t* n, boolean refresh)
```

**Purpose:** Draws the current value of a number widget to the screen, erasing the previous value first.

**Parameters:**
- `n` — The widget to draw.
- `refresh` — If true, force a full redraw even if the value hasn't changed.

**Returns:** Nothing.

**Key Logic:**
1. Reads `*n->num` to get the current value.
2. Handles negative numbers: clamps to -9 or -99 depending on `numdigits`, then negates for rendering.
3. Clears the display area by copying the background region from screen `BG` (screen 4) to screen `FG` (screen 0) using `V_CopyRect`. The region width is `numdigits * patch_width`.
4. The special sentinel value 1994 means "not applicable" — if the number equals 1994, the widget is cleared but no digits are drawn. This is used for weapons that use no ammo (e.g., fists, chainsaw).
5. Draws zero explicitly if the value is 0 (since the loop below only executes while `num != 0`).
6. Draws digits right-to-left using `num % 10` for each digit and `num /= 10` to advance, decrementing `x` by one glyph width per digit.
7. If negative, draws the minus sign 8 pixels left of the leftmost digit.

---

### `STlib_updateNum`

```c
void STlib_updateNum(st_number_t* n, boolean refresh)
```

**Purpose:** Conditionally redraws a number widget if the widget is enabled.

**Parameters:**
- `n` — The widget to update.
- `refresh` — Force redraw flag.

**Returns:** Nothing.

**Key Logic:** Checks `*n->on`; if true, calls `STlib_drawNum`.

---

### `STlib_initPercent`

```c
void STlib_initPercent(st_percent_t* p, int x, int y, patch_t** pl,
                        int* num, boolean* on, patch_t* percent)
```

**Purpose:** Initializes a percentage widget, which is a number widget with an appended percent-sign glyph.

**Parameters:**
- `p` — Pointer to the `st_percent_t` widget to initialize.
- `x`, `y` — Position (right edge, since the embedded number is right-justified).
- `pl` — Array of digit patches for 0-9.
- `num` — Pointer to the value to display.
- `on` — Visibility boolean pointer.
- `percent` — The percent-sign patch to draw to the right of the number.

**Returns:** Nothing.

**Key Logic:** Calls `STlib_initNum` for the embedded number widget with `width=3`, then stores the percent patch pointer.

---

### `STlib_updatePercent`

```c
void STlib_updatePercent(st_percent_t* per, int refresh)
```

**Purpose:** Draws or updates a percent widget (number + percent sign).

**Parameters:**
- `per` — The widget to update.
- `refresh` — Force redraw flag.

**Returns:** Nothing.

**Key Logic:** If `refresh` and the widget is enabled, draws the percent-sign patch at the widget's coordinates. Then calls `STlib_updateNum` to draw/update the number portion.

---

### `STlib_initMultIcon`

```c
void STlib_initMultIcon(st_multicon_t* i, int x, int y,
                         patch_t** il, int* inum, boolean* on)
```

**Purpose:** Initializes a multi-state icon widget that displays one of several patches depending on an integer index.

**Parameters:**
- `i` — Pointer to the `st_multicon_t` widget.
- `x`, `y` — Center position of the icon.
- `il` — Array of patches, indexed by the value at `*inum`.
- `inum` — Pointer to the current icon index value.
- `on` — Visibility boolean pointer.

**Returns:** Nothing.

**Key Logic:** Sets `oldinum = -1` so the first update always triggers a draw (since -1 is the "not yet drawn" sentinel).

---

### `STlib_updateMultIcon`

```c
void STlib_updateMultIcon(st_multicon_t* mi, boolean refresh)
```

**Purpose:** Redraws a multi-state icon if its value has changed or a refresh is requested.

**Parameters:**
- `mi` — The widget to update.
- `refresh` — Force redraw flag.

**Returns:** Nothing.

**Key Logic:**
1. Only acts if `*mi->on` is true, `*mi->inum != -1`, and either the index changed or `refresh` is set.
2. If there was a previously drawn icon (`oldinum != -1`), erases it by copying its bounding box from the background screen using `V_CopyRect`. The old icon's `leftoffset` and `topoffset` are used to compute the erase region.
3. Draws the new icon patch at `(mi->x, mi->y)` using `V_DrawPatch`.
4. Updates `mi->oldinum` to the new index.

---

### `STlib_initBinIcon`

```c
void STlib_initBinIcon(st_binicon_t* b, int x, int y,
                        patch_t* i, boolean* val, boolean* on)
```

**Purpose:** Initializes a binary (on/off) icon widget. The icon is either shown or hidden based on a boolean value.

**Parameters:**
- `b` — Pointer to the `st_binicon_t` widget.
- `x`, `y` — Center position.
- `i` — The patch to display when the value is true.
- `val` — Pointer to the boolean that controls whether the icon is shown.
- `on` — Pointer to a boolean controlling overall widget visibility.

**Returns:** Nothing.

**Key Logic:** Sets `oldval = 0` so the first update triggers a draw.

---

### `STlib_updateBinIcon`

```c
void STlib_updateBinIcon(st_binicon_t* bi, boolean refresh)
```

**Purpose:** Redraws a binary icon if its state has changed.

**Parameters:**
- `bi` — The widget to update.
- `refresh` — Force redraw flag.

**Returns:** Nothing.

**Key Logic:**
1. Only acts if `*bi->on` and the value changed (or `refresh` is set).
2. Computes the bounding box of the icon patch using its `leftoffset`, `topoffset`, `width`, and `height`.
3. If `*bi->val` is true, calls `V_DrawPatch` to show the icon.
4. If false, calls `V_CopyRect` to copy the background over the icon area, effectively hiding it.
5. Updates `bi->oldval`.

---

## Data Structures

All widget structures are declared in `st_lib.h`. See that file's documentation for full field descriptions.

| Structure | Purpose |
|-----------|---------|
| `st_number_t` | Right-justified number display widget |
| `st_percent_t` | Number + percent sign composite widget |
| `st_multicon_t` | Multi-state icon (one of N patches) widget |
| `st_binicon_t` | Binary show/hide icon widget |

---

## Dependencies

| File | Purpose |
|------|---------|
| `doomdef.h` | `SCREENWIDTH`, `SCREENHEIGHT`, `boolean`, `byte` |
| `z_zone.h` | Zone memory (indirectly via included headers) |
| `v_video.h` | `V_DrawPatch`, `V_CopyRect`, `screens[]` |
| `m_swap.h` | `SHORT()` macro for endian-safe patch field access |
| `i_system.h` | `I_Error` for bounds checking |
| `w_wad.h` | `W_CacheLumpName` for loading the minus sign patch |
| `st_stuff.h` | `ST_Y` constant (status bar Y origin on screen) |
| `st_lib.h` | Own header; declares all widget types and function prototypes |
| `r_local.h` | Rendering definitions (pulled in for `patch_t` and related types) |
| `am_map.h` (extern) | `automapactive` boolean used to suppress drawing in automap mode |
