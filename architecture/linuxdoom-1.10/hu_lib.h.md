# File Overview

`hu_lib.h` is the public header for DOOM's heads-up display widget library. It defines the three widget types used to display text during gameplay — `hu_textline_t` (single text line), `hu_stext_t` (scrolling multi-line text), and `hu_itext_t` (interactive input text line) — and declares all the functions that operate on them. Any module that needs to display or interact with HUD text includes this header.

## Global Variables

None defined in this header. It only defines types, constants, and function declarations.

## Data Structures

### `hu_textline_t` — Text Line Widget

```c
typedef struct {
    int       x;                        // Left-justified screen X coordinate
    int       y;                        // Screen Y coordinate
    patch_t** f;                        // Pointer to font patch array
    int       sc;                       // Start character (ASCII code of first font glyph)
    char      l[HU_MAXLINELENGTH+1];   // Text buffer: up to 80 characters + null terminator
    int       len;                      // Current number of characters in the buffer
    int       needsupdate;              // Non-zero = widget must be redrawn or erased
} hu_textline_t;
```

The base widget. Both `hu_stext_t` and `hu_itext_t` contain or are built on `hu_textline_t` instances. The `needsupdate` field counts down from 4 each time `HUlib_eraseTextLine` is called, serving as a short-lived dirty flag.

---

### `hu_stext_t` — Scrolling Text Window Widget

```c
typedef struct {
    hu_textline_t l[HU_MAXLINES]; // Circular ring of up to 4 text lines
    int           h;              // Height: number of lines currently in use
    int           cl;             // Current line index (most recently added)
    boolean*      on;             // Pointer to external boolean controlling visibility
    boolean       laston;         // Previous frame's value of *on, for transition detection
} hu_stext_t;
```

Used for the player message display in the upper-left corner of the screen. Lines are stored in a circular buffer; when a new message arrives, the oldest line is overwritten. Lines are drawn newest-first from the bottom up.

---

### `hu_itext_t` — Input Text Line Widget

```c
typedef struct {
    hu_textline_t l;      // The underlying text line
    int           lm;     // Left margin: index before which characters cannot be deleted
    boolean*      on;     // Pointer to external boolean controlling visibility
    boolean       laston; // Previous frame's value of *on
} hu_itext_t;
```

Used for the multiplayer chat input box. The `lm` (left margin) field marks the end of any fixed prefix (such as `"GREEN: "`) that the user typed into the chat; backspace cannot delete past this point.

## Constants and Macros

| Constant | Value | Purpose |
|----------|-------|---------|
| `BG` | `1` | Background screen buffer index for `V_DrawPatch`. |
| `FG` | `0` | Foreground screen buffer index for `V_DrawPatchDirect`. |
| `HU_CHARERASE` | `KEY_BACKSPACE` | The key used to erase a character in input widgets. |
| `HU_MAXLINES` | `4` | Maximum number of lines in a scrolling text widget. |
| `HU_MAXLINELENGTH` | `80` | Maximum number of characters in a single text line. |

## Functions

### Text Line Functions

```c
void    HUlib_init(void);
void    HUlib_clearTextLine(hu_textline_t *t);
void    HUlib_initTextLine(hu_textline_t *t, int x, int y, patch_t **f, int sc);
boolean HUlib_addCharToTextLine(hu_textline_t *t, char ch);
boolean HUlib_delCharFromTextLine(hu_textline_t *t);
void    HUlib_drawTextLine(hu_textline_t *l, boolean drawcursor);
void    HUlib_eraseTextLine(hu_textline_t *l);
```

### Scrolling Text Widget Functions

```c
void HUlib_initSText(hu_stext_t* s, int x, int y, int h,
                     patch_t** font, int startchar, boolean* on);
void HUlib_addLineToSText(hu_stext_t* s);
void HUlib_addMessageToSText(hu_stext_t* s, char* prefix, char* msg);
void HUlib_drawSText(hu_stext_t* s);
void HUlib_eraseSText(hu_stext_t* s);
```

### Input Text Widget Functions

```c
void    HUlib_initIText(hu_itext_t* it, int x, int y,
                        patch_t** font, int startchar, boolean* on);
void    HUlib_delCharFromIText(hu_itext_t* it);
void    HUlib_eraseLineFromIText(hu_itext_t* it);
void    HUlib_resetIText(hu_itext_t* it);
void    HUlib_addPrefixToIText(hu_itext_t* it, char* str);
boolean HUlib_keyInIText(hu_itext_t* it, unsigned char ch);
void    HUlib_drawIText(hu_itext_t* it);
void    HUlib_eraseIText(hu_itext_t* it);
```

## Dependencies

- `r_defs.h` — Provides the `patch_t` type used by font arrays.
- `doomtype.h` (transitively) — Provides `boolean`.
