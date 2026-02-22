# File Overview

**Source file:** `linuxdoom-1.10/f_wipe.h`

`f_wipe.h` is the public interface header for the screen-wipe transition system implemented in `f_wipe.c`. It defines the available wipe algorithm identifiers and declares the three public API functions that callers use to set up and drive a screen transition.

The header is included by `d_main.c`, which is the primary caller of the wipe system: it calls `wipe_StartScreen` before rendering the new state, then `wipe_EndScreen` after, and finally calls `wipe_ScreenWipe` each render frame until the transition completes.

---

## Global Variables

None. This header declares no variables.

---

## Data Structures

### Wipe algorithm enumeration (anonymous `enum`)

```c
enum {
    wipe_ColorXForm,  // = 0
    wipe_Melt,        // = 1
    wipe_NUMWIPES     // = 2  (sentinel / count)
};
```

| Enumerator | Value | Description |
|------------|-------|-------------|
| `wipe_ColorXForm` | 0 | Per-pixel palette-index fade. Each pixel steps from its start palette index toward the corresponding end index by a fixed step per tic. Only meaningful in 8-bit paletted mode. |
| `wipe_Melt` | 1 | The iconic DOOM melt effect. Screen columns fall downward at slightly randomized start times and acceleration, revealing the new screen beneath the old. |
| `wipe_NUMWIPES` | 2 | Total count of wipe algorithms; used for bounds checking. |

The `wipeno` parameter of `wipe_ScreenWipe` is expected to be one of `wipe_ColorXForm` or `wipe_Melt`.

---

## Functions

### `wipe_StartScreen`

```c
int wipe_StartScreen(int x, int y, int width, int height);
```

**Purpose:** Captures a snapshot of the current display into the "start" buffer (`screens[2]`). Must be called before the new game state is rendered, so that the wipe system has the "before" image to transition from.

**Parameters:**
- `x`, `y` — origin of the capture region (currently unused; always captures full screen)
- `width`, `height` — dimensions of the capture region (currently unused)

**Returns:** 0 on success.

**Calling convention:** Called once, before drawing the new game state.

---

### `wipe_EndScreen`

```c
int wipe_EndScreen(int x, int y, int width, int height);
```

**Purpose:** Captures the newly-rendered game state into the "end" buffer (`screens[3]`), then restores the start image to the primary screen buffer so the wipe animation begins from the correct starting point.

**Parameters:**
- `x`, `y` — origin of the region (passed to `V_DrawBlock` for restoration)
- `width`, `height` — dimensions of the region

**Returns:** 0 on success.

**Calling convention:** Called once, immediately after the new game state has been fully rendered.

---

### `wipe_ScreenWipe`

```c
int wipe_ScreenWipe(int wipeno, int x, int y, int width, int height, int ticks);
```

**Purpose:** Advances the wipe animation by `ticks` steps and renders the current interpolated frame to `screens[0]`. Handles first-call initialization and final cleanup automatically via an internal `go` flag.

**Parameters:**
- `wipeno` — algorithm selector: `wipe_ColorXForm` (0) or `wipe_Melt` (1)
- `x`, `y` — origin of the wipe region (passed to `V_MarkRect`)
- `width`, `height` — dimensions of the wipe region
- `ticks` — number of animation sub-steps to advance per call (usually 1)

**Returns:** 1 when the wipe is complete and the caller should stop calling; 0 while still animating.

**Calling convention:** Called every render frame (in the `D_Display` loop) after `wipe_StartScreen` and `wipe_EndScreen` have been called. The caller loops until this function returns 1.

---

## Dependencies

| Header | Reason |
|--------|--------|
| *(none)* | This header is self-contained; it uses only primitive C types (`int`) in all declarations |

Consumers of this header typically also include `doomdef.h` for `SCREENWIDTH`/`SCREENHEIGHT`, but those constants are not required by the header itself.
