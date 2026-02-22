# File Overview

**Source file:** `linuxdoom-1.10/f_wipe.c`

`f_wipe.c` implements the **screen wipe / transition effect** that plays between game states (e.g., from the title screen into a level, or between levels). The system captures two screen snapshots — the "start" screen (what was showing) and the "end" screen (what should appear next) — and then interpolates between them over several tics to create a visual transition.

Two wipe algorithms are implemented:

1. **ColorXForm** (`wipe_ColorXForm`): A simple per-pixel palette-index fade. Each pixel of the start screen steps toward the matching pixel in the end screen by `ticks` palette-index steps per call. Pure 8-bit effect; not commonly used in the shipped game.

2. **Melt** (`wipe_Melt`): The iconic DOOM "melt" effect. Screen columns fall downward at variable speeds (slightly randomized so each column starts at a different time), revealing the new screen beneath the old one as it slides away.

The module is self-contained. It operates directly on the `screens[]` framebuffer array from `v_video.c` and uses the zone allocator for temporary memory.

---

## Global Variables

### Static (module-private)

| Type | Name | Purpose |
|------|------|---------|
| `static boolean` | `go` | State flag: `1` while a wipe is in progress, `0` when complete. Used by `wipe_ScreenWipe` to initialize the effect on first call and finalize it when done |
| `static byte*` | `wipe_scr_start` | Pointer to the "start" framebuffer — the screen contents before the transition. Set to `screens[2]` |
| `static byte*` | `wipe_scr_end` | Pointer to the "end" framebuffer — the screen contents of the new game state. Set to `screens[3]` |
| `static byte*` | `wipe_scr` | Pointer to the working/output framebuffer — the screen being rendered during the wipe. Set to `screens[0]` (the primary display buffer) |
| `static int*` | `y` | Array of per-column vertical offsets used by the Melt algorithm. Allocated from the zone heap in `wipe_initMelt`, freed in `wipe_exitMelt`. Negative values mean "not ready to scroll yet" |

---

## Functions

### `wipe_shittyColMajorXform`

```c
void wipe_shittyColMajorXform(short* array, int width, int height)
```

**Purpose:** Transposes a row-major pixel buffer into column-major order, treating each pair of bytes as a `short`. This reordering makes the Melt algorithm's inner loop more cache-friendly because the algorithm accesses pixels column by column.

**Parameters:**
- `array` — the buffer to transform in-place (width × height `short` values, i.e., width×2 bytes per pixel pair)
- `width` — number of columns (in `short` units, i.e., `SCREENWIDTH / 2`)
- `height` — number of rows (`SCREENHEIGHT`)

**Key logic:** Allocates a temporary buffer via `Z_Malloc(width * height * 2, PU_STATIC, 0)`, copies the transposed data, copies it back, then frees the temporary.

**Returns:** void

**Note:** The name reflects the original developers' opinion of the approach; it is effective despite the colorful label.

---

### `wipe_initColorXForm`

```c
int wipe_initColorXForm(int width, int height, int ticks)
```

**Purpose:** Initializes the ColorXForm wipe by copying `wipe_scr_start` into the working screen `wipe_scr`.

**Parameters:**
- `width`, `height` — screen dimensions
- `ticks` — unused during initialization

**Returns:** 0 (success; never signals completion).

---

### `wipe_doColorXForm`

```c
int wipe_doColorXForm(int width, int height, int ticks)
```

**Purpose:** Advances the ColorXForm wipe by one step. Each pixel in `wipe_scr` is moved `ticks` palette-index steps toward the corresponding pixel in `wipe_scr_end`. Pixels already at their target are left unchanged.

**Parameters:**
- `width`, `height` — screen dimensions (together define the total pixel count)
- `ticks` — the step size (palette-index units) per call

**Key logic:**
- Walks both `wipe_scr` (write pointer `w`) and `wipe_scr_end` (read pointer `e`) in lockstep.
- For each pixel: if `*w > *e`, subtracts `ticks`, clamping to `*e`; if `*w < *e`, adds `ticks`, clamping to `*e`.
- Sets `changed = true` whenever any pixel is still transitioning.

**Returns:** `!changed` — returns 1 (done) when all pixels have reached their target, 0 while still animating.

---

### `wipe_exitColorXForm`

```c
int wipe_exitColorXForm(int width, int height, int ticks)
```

**Purpose:** Cleanup for the ColorXForm wipe. No resources to free; this is a no-op.

**Returns:** 0.

---

### `wipe_initMelt`

```c
int wipe_initMelt(int width, int height, int ticks)
```

**Purpose:** Initializes the Melt wipe effect.

**Key logic:**
1. Copies `wipe_scr_start` into `wipe_scr` (so the display starts showing the old screen).
2. Transforms both `wipe_scr_start` and `wipe_scr_end` to column-major order via `wipe_shittyColMajorXform`. Width passed is `width/2` because the data is treated as `short` pairs.
3. Allocates the per-column `y[]` array (`width` ints) from the zone heap.
4. Seeds column positions: `y[0] = -(M_Random() % 16)` (random start delay in [-15, 0]). Each subsequent column is `y[i] = y[i-1] + r` where `r = (M_Random() % 3) - 1`, i.e., ±1 or 0. Values are clamped to `[−15, 0]`.

**Returns:** 0.

---

### `wipe_doMelt`

```c
int wipe_doMelt(int width, int height, int ticks)
```

**Purpose:** Advances the Melt animation by `ticks` sub-steps. For each tick, each column either increments its countdown (if still negative) or draws a slice of the new screen over the old screen column.

**Parameters:**
- `width` — screen width; halved internally (`width /= 2`) for `short`-pair indexing
- `height` — screen height
- `ticks` — number of sub-steps to apply this call (typically 1)

**Key logic (per tick, per column):**
- If `y[i] < 0`: increment (move toward 0), set `done = false`.
- If `y[i] < height`:
  - `dy = (y[i] < 16) ? y[i] + 1 : 8` — columns accelerate during the first 16 pixels, then fall at a fixed 8-pixel-per-tic rate.
  - Clamps `dy` so the column does not overshoot `height`.
  - Copies `dy` rows from the column-major `wipe_scr_end` into the row-major `wipe_scr` at the current y position. The column-major indexing uses `i * height + y[i]` as the source and `y[i] * width + i` as the destination.
  - Fills the remainder of the column below `y[i]` from `wipe_scr_start` (the old screen continues to show above the melt front).
  - Advances `y[i] += dy`; sets `done = false`.

**Returns:** `done` — `true` (1) when all columns have reached the bottom, `false` (0) otherwise.

---

### `wipe_exitMelt`

```c
int wipe_exitMelt(int width, int height, int ticks)
```

**Purpose:** Frees the per-column `y[]` array allocated during `wipe_initMelt`.

**Returns:** 0.

---

### `wipe_StartScreen`

```c
int wipe_StartScreen(int x, int y, int width, int height)
```

**Purpose:** Captures the current display into `wipe_scr_start` (which is `screens[2]`). Called before the new game state is rendered, to snapshot the "before" image.

**Parameters:** `x`, `y`, `width`, `height` — currently ignored; always captures the full screen.

**Key logic:** Sets `wipe_scr_start = screens[2]`, then calls `I_ReadScreen(wipe_scr_start)` to copy the hardware framebuffer into the buffer.

**Returns:** 0.

---

### `wipe_EndScreen`

```c
int wipe_EndScreen(int x, int y, int width, int height)
```

**Purpose:** Captures the freshly-rendered new game state into `wipe_scr_end` (which is `screens[3]`), then restores the start screen to the display so the wipe begins from the correct image.

**Parameters:** `x`, `y`, `width`, `height` — the region to capture (passed through to `V_DrawBlock`).

**Key logic:**
1. Sets `wipe_scr_end = screens[3]`.
2. Calls `I_ReadScreen(wipe_scr_end)` to snapshot the new state.
3. Calls `V_DrawBlock(x, y, 0, width, height, wipe_scr_start)` to restore the start screen to the display buffer, so the first wipe frame shows the correct old image.

**Returns:** 0.

---

### `wipe_ScreenWipe`

```c
int wipe_ScreenWipe(int wipeno, int x, int y, int width, int height, int ticks)
```

**Purpose:** The main per-frame wipe driver. Selects the wipe algorithm by index, calls its init/do/exit functions through a static function-pointer table, and signals completion.

**Parameters:**
- `wipeno` — wipe algorithm index: 0 = ColorXForm, 1 = Melt (values from the `enum` in `f_wipe.h`)
- `x`, `y` — origin of the wipe region (currently unused beyond passing to `V_MarkRect`)
- `width`, `height` — dimensions of the wipe region
- `ticks` — number of animation sub-steps to advance this call

**Key logic:**

The static function table `wipes[]` is organized as init/do/exit triplets:
```c
static int (*wipes[])(int, int, int) = {
    wipe_initColorXForm, wipe_doColorXForm, wipe_exitColorXForm,   // wipeno=0
    wipe_initMelt,       wipe_doMelt,       wipe_exitMelt           // wipeno=1
};
```

- On first call (`!go`): sets `go = 1`, sets `wipe_scr = screens[0]`, calls `wipes[wipeno*3]` (init).
- Every call: calls `V_MarkRect(0, 0, width, height)` to dirty the full screen, then calls `wipes[wipeno*3+1]` (do step). The return value `rc` is non-zero when the wipe is complete.
- When `rc` is non-zero: sets `go = 0`, calls `wipes[wipeno*3+2]` (exit/cleanup).

**Returns:** `!go` — returns 1 when the wipe is finished and `go` has been cleared, 0 while still animating.

---

## Data Structures

No structs or typedefs are defined in this file. The wipe algorithm dispatch is handled via a static array of function pointers inside `wipe_ScreenWipe`.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `z_zone.h` | `Z_Malloc` / `Z_Free` for temporary column-offset array (`y[]`) and the column-major transform buffer |
| `i_video.h` | `I_ReadScreen` to capture the hardware framebuffer into the start/end screen buffers |
| `v_video.h` | `V_DrawBlock` (restore start screen after `wipe_EndScreen`), `V_MarkRect` (mark dirty region), `screens[]` array |
| `m_random.h` | `M_Random()` to generate random per-column start offsets in `wipe_initMelt` |
| `doomdef.h` | Basic DOOM type definitions and constants (`SCREENWIDTH`, `SCREENHEIGHT`, `boolean`) |
| `f_wipe.h` | Self-referential: the wipe enum and public API declarations |
