# File Overview

`st_lib.h` is the header for the status bar widget library. It defines the four widget data structures used by the status bar system and declares all public initialization and update functions. This header is included by `st_stuff.c` (the main status bar controller) to create, configure, and update the various display elements that make up DOOM's HUD.

The widget design follows a consistent pattern: each widget holds pointers to live game data rather than cached copies. The update functions are called every game tick and only redraw when the underlying data has changed, minimizing framebuffer writes.

Two screen buffer constants `BG` and `FG` are defined here, reflecting the two-buffer scheme: `BG` (screen 4) holds a static copy of the status bar background used for erasing, and `FG` (screen 0) is the active display framebuffer.

---

## Global Variables

None. This is a pure declaration header.

---

## Data Structures

### `st_number_t` — Number Widget

Displays a right-justified integer in a fixed-width digit field.

```c
typedef struct
{
    int       x;        // X coordinate of the right edge of the number
    int       y;        // Y coordinate
    int       width;    // Maximum number of digits to display
    int       oldnum;   // The previously displayed value (for change detection)
    int*      num;      // Pointer to the current value to display
    boolean*  on;       // Pointer to visibility flag; widget draws only when *on is true
    patch_t** p;        // Array of 10 patches for digit glyphs 0-9
    int       data;     // User-defined extra data (used by st_stuff.c to track weapon type)
} st_number_t;
```

**Usage:** Used for ammo count, health, armor, frags, and ammo-per-weapon-type displays.

**Special sentinel:** If `*num == 1994`, the widget displays nothing (used for weapons with no ammo).

---

### `st_percent_t` — Percent Widget

A composite widget containing a `st_number_t` plus a percent-sign glyph drawn to its right.

```c
typedef struct
{
    st_number_t  n;   // Embedded number widget (initialized with width=3)
    patch_t*     p;   // The percent sign patch (STTPRCNT lump)
} st_percent_t;
```

**Usage:** Used for the health and armor percentage readouts on the status bar.

---

### `st_multicon_t` — Multiple Icon Widget

Displays one of several icon patches based on an integer index. Used for widgets where the display can be in one of N discrete states.

```c
typedef struct
{
    int       x;        // Center X of the icon
    int       y;        // Center Y of the icon
    int       oldinum;  // Previously displayed icon index (-1 = none drawn yet)
    int*      inum;     // Pointer to the current icon index (-1 = draw nothing)
    boolean*  on;       // Visibility flag pointer
    patch_t** p;        // Array of patches, one per possible state
    int       data;     // User-defined extra data
} st_multicon_t;
```

**Usage:** Used for the face/status indicator (which cycles through ~40 face states), the keycard slots (blue/yellow/red card or skull, or empty), and weapon ownership indicators (gray for unowned, yellow for owned).

---

### `st_binicon_t` — Binary Icon Widget

Displays a single patch when a boolean is true, or erases it when false.

```c
typedef struct
{
    int       x;       // Center X of the icon
    int       y;       // Center Y of the icon
    int       oldval;  // Previously displayed state (for change detection)
    boolean*  val;     // Pointer to the boolean state controlling display
    boolean*  on;      // Overall visibility flag pointer
    patch_t*  p;       // The icon patch to show when *val is true
    int       data;    // User-defined extra data
} st_binicon_t;
```

**Usage:** Used for the `STARMS` (weapons panel background) display — shown only in non-deathmatch games.

---

## Constants

```c
#define BG  4   // Screen buffer index for status bar background (erase source)
#define FG  0   // Screen buffer index for the active display framebuffer
```

---

## Functions

### `STlib_init`

```c
void STlib_init(void);
```

Loads the `STTMINUS` patch for negative number display.

---

### `STlib_initNum`

```c
void STlib_initNum(st_number_t* n, int x, int y, patch_t** pl,
                   int* num, boolean* on, int width);
```

Initializes a number widget. See `st_lib.c.md` for detailed parameter descriptions.

---

### `STlib_updateNum`

```c
void STlib_updateNum(st_number_t* n, boolean refresh);
```

Conditionally redraws a number widget. Checks `*n->on` before drawing.

---

### `STlib_initPercent`

```c
void STlib_initPercent(st_percent_t* p, int x, int y, patch_t** pl,
                        int* num, boolean* on, patch_t* percent);
```

Initializes a percent widget (number + `%` glyph).

---

### `STlib_updatePercent`

```c
void STlib_updatePercent(st_percent_t* per, int refresh);
```

Redraws a percent widget if enabled.

---

### `STlib_initMultIcon`

```c
void STlib_initMultIcon(st_multicon_t* mi, int x, int y,
                         patch_t** il, int* inum, boolean* on);
```

Initializes a multi-state icon widget.

---

### `STlib_updateMultIcon`

```c
void STlib_updateMultIcon(st_multicon_t* mi, boolean refresh);
```

Redraws a multi-state icon if its index has changed.

---

### `STlib_initBinIcon`

```c
void STlib_initBinIcon(st_binicon_t* b, int x, int y,
                        patch_t* i, boolean* val, boolean* on);
```

Initializes a binary show/hide icon widget.

---

### `STlib_updateBinIcon`

```c
void STlib_updateBinIcon(st_binicon_t* bi, boolean refresh);
```

Shows or hides a binary icon based on its boolean state.

---

## Dependencies

| File | Purpose |
|------|---------|
| `r_defs.h` | `patch_t` structure used by all widget types for patch pointers |
