# File Overview

`hu_lib.c` implements the low-level heads-up display (HUD) widget library for DOOM. It provides the foundational building blocks for all on-screen text display during gameplay: single-line text display, scrolling text windows, and interactive text input. Every HUD element — player messages, the map title, and the multiplayer chat input — is built from these primitives. The library handles drawing text using the game's patch-based font system, erasing old text from the screen border areas, and managing the update state of each widget.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `extern boolean` | `automapactive` | Imported from `am_map.c`. Used by `HUlib_eraseTextLine` to determine whether the automap is showing, which changes the erase logic. |

### Macro Definitions

| Macro | Expansion | Purpose |
|-------|-----------|---------|
| `noterased` | `viewwindowx` | Repurposes the renderer's `viewwindowx` variable as a boolean flag indicating whether the screen has been erased. Non-zero means it has NOT been erased. |
| `BG` | `1` | Background screen buffer index. |
| `FG` | `0` | Foreground screen buffer index used when drawing patches. |

## Functions

### `HUlib_init`

```c
void HUlib_init(void)
```

**Purpose:** Placeholder initialization function for the HU widget library. Currently empty — all initialization happens per-widget.

**Parameters:** None.
**Returns:** Nothing.

---

### `HUlib_clearTextLine`

```c
void HUlib_clearTextLine(hu_textline_t* t)
```

**Purpose:** Resets a text line widget to empty, setting length to zero, null-terminating the buffer, and marking the widget as needing a redraw.

**Parameters:**
- `t` — Pointer to the `hu_textline_t` to clear.

**Returns:** Nothing.

**Key logic:** Sets `t->len = 0`, `t->l[0] = 0`, and `t->needsupdate = true`.

---

### `HUlib_initTextLine`

```c
void HUlib_initTextLine(hu_textline_t* t, int x, int y, patch_t** f, int sc)
```

**Purpose:** Initializes a text line widget with position, font, and starting character, then clears it to empty.

**Parameters:**
- `t` — Pointer to the `hu_textline_t` to initialize.
- `x` — Screen X coordinate (left edge) for drawing.
- `y` — Screen Y coordinate (top edge) for drawing.
- `f` — Array of pointers to font patches.
- `sc` — Start character (ASCII code of the first character in the font array).

**Returns:** Nothing.

---

### `HUlib_addCharToTextLine`

```c
boolean HUlib_addCharToTextLine(hu_textline_t* t, char ch)
```

**Purpose:** Appends a single character to a text line widget's buffer if space is available.

**Parameters:**
- `t` — Pointer to the text line.
- `ch` — Character to append.

**Returns:** `true` if the character was added; `false` if the line was already at `HU_MAXLINELENGTH` (80 characters).

**Key logic:** Appends `ch` to `t->l[t->len]`, increments `len`, null-terminates, and sets `t->needsupdate = 4`.

---

### `HUlib_delCharFromTextLine`

```c
boolean HUlib_delCharFromTextLine(hu_textline_t* t)
```

**Purpose:** Removes the last character from a text line widget.

**Parameters:**
- `t` — Pointer to the text line.

**Returns:** `true` if a character was deleted; `false` if the line was already empty.

---

### `HUlib_drawTextLine`

```c
void HUlib_drawTextLine(hu_textline_t* l, boolean drawcursor)
```

**Purpose:** Renders a text line widget to the screen using the game's font patches, optionally drawing a cursor character ('_') at the end.

**Parameters:**
- `l` — Pointer to the text line to draw.
- `drawcursor` — If `true`, draws an underscore cursor after the last character.

**Returns:** Nothing.

**Key logic:**
- Iterates over each character in `l->l[]`, converts to uppercase via `toupper`.
- Non-space characters in the range `[l->sc .. '_']` are drawn using `V_DrawPatchDirect` with the appropriate font patch from `l->f[]`.
- Space characters advance the cursor position by 4 pixels.
- Drawing stops when the next character would exceed `SCREENWIDTH`.
- The cursor is drawn if `drawcursor` is true and the cursor patch fits on screen.

---

### `HUlib_eraseTextLine`

```c
void HUlib_eraseTextLine(hu_textline_t* l)
```

**Purpose:** Erases the background behind a text line widget, restoring the status bar or border graphics. Only operates when the automap is inactive, the view window is reduced (border visible), and the widget needs updating.

**Parameters:**
- `l` — Pointer to the text line to erase.

**Returns:** Nothing.

**Key logic:**
- Only erases when `!automapactive && viewwindowx && l->needsupdate`.
- Calculates line height from the first font patch.
- For each row of pixels in the text line's vertical span: if the row is above or below the view window, the entire row is erased via `R_VideoErase`; otherwise only the left and right border strips are erased.
- Decrements `l->needsupdate` toward zero each call.

---

### `HUlib_initSText`

```c
void HUlib_initSText(hu_stext_t* s, int x, int y, int h, patch_t** font, int startchar, boolean* on)
```

**Purpose:** Initializes a scrolling text widget, which is a stack of `h` text lines arranged vertically. Lines are stacked upward from `y`.

**Parameters:**
- `s` — Pointer to the `hu_stext_t` to initialize.
- `x`, `y` — Screen position of the bottommost line.
- `h` — Number of lines in the widget (max `HU_MAXLINES` = 4).
- `font` — Font patch array.
- `startchar` — First character in the font array.
- `on` — Pointer to a boolean controlling widget visibility.

**Returns:** Nothing.

---

### `HUlib_addLineToSText`

```c
void HUlib_addLineToSText(hu_stext_t* s)
```

**Purpose:** Advances the scrolling text widget to the next line slot (circular buffer), clears it, and marks all lines for redraw.

**Parameters:**
- `s` — Pointer to the scrolling text widget.

**Returns:** Nothing.

---

### `HUlib_addMessageToSText`

```c
void HUlib_addMessageToSText(hu_stext_t* s, char* prefix, char* msg)
```

**Purpose:** Adds a new message (with optional prefix) to the scrolling text widget by advancing the line and copying characters.

**Parameters:**
- `s` — Pointer to the scrolling text widget.
- `prefix` — Optional prefix string (e.g., player name). May be NULL.
- `msg` — The message string to add.

**Returns:** Nothing.

---

### `HUlib_drawSText`

```c
void HUlib_drawSText(hu_stext_t* s)
```

**Purpose:** Draws all visible lines of a scrolling text widget, from most recent to oldest (top to bottom on screen). Does nothing if `*s->on` is false.

**Parameters:**
- `s` — Pointer to the scrolling text widget.

**Returns:** Nothing.

---

### `HUlib_eraseSText`

```c
void HUlib_eraseSText(hu_stext_t* s)
```

**Purpose:** Erases all lines of a scrolling text widget. If the widget just became invisible (`laston && !*on`), marks all lines as needing erase.

**Parameters:**
- `s` — Pointer to the scrolling text widget.

**Returns:** Nothing.

---

### `HUlib_initIText`

```c
void HUlib_initIText(hu_itext_t* it, int x, int y, patch_t** font, int startchar, boolean* on)
```

**Purpose:** Initializes an input text widget, which is a single text line with a left margin and cursor support.

**Parameters:**
- `it` — Pointer to the `hu_itext_t` to initialize.
- `x`, `y` — Screen position.
- `font` — Font patch array.
- `startchar` — First character in the font array.
- `on` — Pointer to visibility boolean.

**Returns:** Nothing.

---

### `HUlib_delCharFromIText`

```c
void HUlib_delCharFromIText(hu_itext_t* it)
```

**Purpose:** Deletes the last character from an input text widget, respecting the left margin (will not delete past the margin).

**Parameters:**
- `it` — Pointer to the input text widget.

**Returns:** Nothing.

---

### `HUlib_eraseLineFromIText`

```c
void HUlib_eraseLineFromIText(hu_itext_t* it)
```

**Purpose:** Deletes all user-typed characters back to the left margin.

**Parameters:**
- `it` — Pointer to the input text widget.

**Returns:** Nothing.

---

### `HUlib_resetIText`

```c
void HUlib_resetIText(hu_itext_t* it)
```

**Purpose:** Resets an input text widget to fully empty, clearing both the text buffer and the left margin.

**Parameters:**
- `it` — Pointer to the input text widget.

**Returns:** Nothing.

---

### `HUlib_addPrefixToIText`

```c
void HUlib_addPrefixToIText(hu_itext_t* it, char* str)
```

**Purpose:** Adds a prefix string to an input text widget and advances the left margin to protect it from deletion.

**Parameters:**
- `it` — Pointer to the input text widget.
- `str` — The prefix string to add.

**Returns:** Nothing.

---

### `HUlib_keyInIText`

```c
boolean HUlib_keyInIText(hu_itext_t* it, unsigned char ch)
```

**Purpose:** Handles keyboard input for an input text widget. Printable characters are added; backspace deletes; enter is accepted but does not modify the buffer (caller handles the result). Other keys are ignored and the function returns false.

**Parameters:**
- `it` — Pointer to the input text widget.
- `ch` — The key character to process.

**Returns:** `true` if the key was consumed; `false` if it was unrecognized.

---

### `HUlib_drawIText`

```c
void HUlib_drawIText(hu_itext_t* it)
```

**Purpose:** Draws an input text widget with its cursor. Does nothing if `*it->on` is false.

**Parameters:**
- `it` — Pointer to the input text widget.

**Returns:** Nothing.

---

### `HUlib_eraseIText`

```c
void HUlib_eraseIText(hu_itext_t* it)
```

**Purpose:** Erases an input text widget from the screen. If the widget just became invisible, marks the underlying text line for erase.

**Parameters:**
- `it` — Pointer to the input text widget.

**Returns:** Nothing.

## Data Structures

Defined in `hu_lib.h` and used throughout this file:

### `hu_textline_t`

```c
typedef struct {
    int       x;               // Left-justified screen X position
    int       y;               // Screen Y position
    patch_t** f;               // Font patch array
    int       sc;              // Start character (ASCII of first font patch)
    char      l[HU_MAXLINELENGTH+1]; // Text buffer (80 chars + null)
    int       len;             // Current text length
    int       needsupdate;     // Non-zero = widget needs redraw/erase
} hu_textline_t;
```

Base widget for all HUD text display.

### `hu_stext_t`

```c
typedef struct {
    hu_textline_t l[HU_MAXLINES]; // Circular buffer of up to 4 text lines
    int           h;              // Number of lines
    int           cl;             // Current (most recent) line index
    boolean*      on;             // Pointer to visibility flag
    boolean       laston;         // Previous visibility state
} hu_stext_t;
```

Scrolling text display widget. Used for player messages.

### `hu_itext_t`

```c
typedef struct {
    hu_textline_t l;      // The single text line
    int           lm;     // Left margin: characters below this index cannot be deleted
    boolean*      on;     // Pointer to visibility flag
    boolean       laston; // Previous visibility state
} hu_itext_t;
```

Interactive text input widget. Used for the multiplayer chat input line.

## Dependencies

- `doomdef.h` — Core DOOM definitions (`boolean`, screen constants).
- `v_video.h` / `v_video.c` — `V_DrawPatchDirect` for rendering font patches.
- `m_swap.h` — `SHORT` macro for endianness-safe patch width/height access.
- `hu_lib.h` — Own header (widget type definitions and function declarations).
- `r_local.h` — `viewwindowx`, `viewwindowy`, `viewwidth`, `viewheight` (view window geometry).
- `r_draw.h` / `r_draw.c` — `R_VideoErase` for erasing screen regions.
- `am_map.c` — `automapactive` (external boolean, controls erase behavior).
