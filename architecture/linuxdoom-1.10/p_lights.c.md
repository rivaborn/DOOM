# File Overview

`p_lights.c` implements all sector-based dynamic lighting effects in the DOOM engine. Every light effect in the game is realized as a thinker - a small data structure registered in the global thinker list - whose associated think function is called once per game tic to update the light level of a sector.

The file provides five distinct lighting behaviors:

1. **Fire flicker** (`fireflicker_t`) - random downward flickers simulating open flame.
2. **Light flash** (`lightflash_t`) - random on/off strobing with independent min/max durations.
3. **Strobe flash** (`strobe_t`) - regular, predictable on/off strobing with configurable dark and bright times.
4. **Glowing light** (`glow_t`) - a smooth oscillation between a minimum and maximum light level.
5. **Instant light changes** (`EV_TurnTagLightsOff`, `EV_LightTurnOn`) - triggered by line specials that set all tagged sectors to a given level immediately.

The strobe and glow effects are spawned during level load from sector specials (via `P_SpawnSpecials` in `p_spec.c`). Strobe effects can also be triggered at runtime by linedef activations. The light level changes are written directly into `sector->lightlevel` each tic, which the renderer reads when computing wall and floor shading.

---

## Global Variables

None. All state for active light effects is stored inside the dynamically allocated thinker structures themselves.

---

## Functions

### `T_FireFlicker`

```c
void T_FireFlicker(fireflicker_t* flick)
```

**Purpose:** Per-tic thinker for the fire flicker effect. Waits a fixed number of tics then randomly drops the light level by a multiple of 16, clamped at `minlight`.

**Parameters:**
- `flick` - Pointer to the fireflicker thinker state for this sector.

**Key logic:**
- Decrements `flick->count` and returns immediately if it has not yet reached zero (the counter resets to 4 each update, giving updates every 4 tics / ~8.6 Hz).
- Computes `amount = (P_Random()&3)*16` - a value of 0, 16, 32, or 48.
- If `sector->lightlevel - amount < minlight`, clamps to `minlight`; otherwise sets `sector->lightlevel = maxlight - amount`.
- Resets `count = 4`.

---

### `P_SpawnFireFlicker`

```c
void P_SpawnFireFlicker(sector_t* sector)
```

**Purpose:** Initializes and registers a fire flicker thinker for the given sector during level load.

**Parameters:**
- `sector` - The sector to apply the effect to.

**Key logic:**
- Zeroes `sector->special` to prevent repeated spawning.
- Allocates a `fireflicker_t` from the zone heap with `PU_LEVSPEC` tag (freed when the level ends).
- Registers it with `P_AddThinker`.
- Sets `maxlight` to the sector's current light level and `minlight` to `P_FindMinSurroundingLight(sector, sector->lightlevel) + 16` (ensures a visible flicker range).
- Sets initial `count = 4`.

---

### `T_LightFlash`

```c
void T_LightFlash(lightflash_t* flash)
```

**Purpose:** Per-tic thinker for broken/random light flashing. Toggles the sector light level between `maxlight` and `minlight` with randomized durations for each phase.

**Parameters:**
- `flash` - Pointer to the lightflash thinker state.

**Key logic:**
- Decrements `flash->count`; returns if not zero.
- If currently at `maxlight`, switches to `minlight` and sets `count = (P_Random()&mintime)+1`.
- If currently at `minlight`, switches to `maxlight` and sets `count = (P_Random()&maxtime)+1`.
- `mintime` (7) and `maxtime` (64) define how long each phase lasts in tics.

---

### `P_SpawnLightFlash`

```c
void P_SpawnLightFlash(sector_t* sector)
```

**Purpose:** Initializes and registers a random light flash thinker during level load.

**Parameters:**
- `sector` - The sector to affect.

**Key logic:**
- Zeroes `sector->special`.
- Sets `maxlight = sector->lightlevel`, `minlight = P_FindMinSurroundingLight(...)`.
- `maxtime = 64`, `mintime = 7`.
- Initial `count = (P_Random()&maxtime)+1` - randomizes initial phase offset so nearby flash sectors do not synchronize.

---

### `T_StrobeFlash`

```c
void T_StrobeFlash(strobe_t* flash)
```

**Purpose:** Per-tic thinker for the strobe light effect. Alternates the sector between `minlight` and `maxlight` using fixed predetermined durations.

**Parameters:**
- `flash` - Pointer to the strobe thinker state.

**Key logic:**
- Decrements `count`; returns if not zero.
- If currently at `minlight`, switches to `maxlight` and sets `count = brighttime` (always `STROBEBRIGHT`, defined in `p_spec.h` as 5 tics).
- If currently at `maxlight`, switches to `minlight` and sets `count = darktime` (either `SLOWDARK` or `FASTDARK`).

---

### `P_SpawnStrobeFlash`

```c
void P_SpawnStrobeFlash(sector_t* sector, int fastOrSlow, int inSync)
```

**Purpose:** Initializes and registers a strobe flash thinker. Called both from `P_SpawnSpecials` (sector type) and `EV_StartLightStrobing` (linedef trigger).

**Parameters:**
- `sector` - The sector to affect.
- `fastOrSlow` - The dark-phase duration in tics. Typical values: `SLOWDARK` (35 tics = 1 second) or `FASTDARK` (15 tics).
- `inSync` - If non-zero, all strobes start in phase (count = 1). If zero, each strobe gets a random starting offset (`(P_Random()&7)+1`).

**Key logic:**
- Sets `darktime = fastOrSlow`, `brighttime = STROBEBRIGHT`.
- `maxlight = sector->lightlevel`, `minlight = P_FindMinSurroundingLight(...)`. If `minlight == maxlight`, forces `minlight = 0` to ensure a visible effect.
- Zeroes `sector->special` after spawning.

---

### `EV_StartLightStrobing`

```c
void EV_StartLightStrobing(line_t* line)
```

**Purpose:** Linedef activation event that starts slow strobe flashing in all sectors tagged with the triggering line's tag.

**Parameters:**
- `line` - The triggering linedef. Its `tag` field identifies which sectors to affect.

**Key logic:**
- Iterates sectors via `P_FindSectorFromLineTag`. Skips any sector that already has `specialdata` set (i.e., is already running a special effect).
- Calls `P_SpawnStrobeFlash(sec, SLOWDARK, 0)` for each matching sector (unsynchronized slow strobe).

---

### `EV_TurnTagLightsOff`

```c
void EV_TurnTagLightsOff(line_t* line)
```

**Purpose:** Linedef activation event that sets all tagged sectors to the minimum light level found among their neighboring sectors.

**Parameters:**
- `line` - The triggering linedef.

**Key logic:**
- Iterates all sectors; for those matching `line->tag`, finds the minimum `lightlevel` among all neighboring sectors by walking the sector's line list and calling `getNextSector`.
- Sets `sector->lightlevel = min`. This is an instantaneous, permanent change (no thinker is spawned).

---

### `EV_LightTurnOn`

```c
void EV_LightTurnOn(line_t* line, int bright)
```

**Purpose:** Linedef activation event that sets all tagged sectors to a specified light level. If `bright` is 0, the brightest neighboring sector's level is used instead.

**Parameters:**
- `line` - The triggering linedef.
- `bright` - Target light level. `0` means "find the highest surrounding light level".

**Key logic:**
- For each sector matching `line->tag`: if `bright == 0`, walks neighboring sectors to find the maximum `lightlevel` and uses that. Otherwise uses the passed `bright` value directly.
- Sets `sector->lightlevel` immediately with no thinker.

---

### `T_Glow`

```c
void T_Glow(glow_t* g)
```

**Purpose:** Per-tic thinker for the smooth glowing light effect. Moves the sector's light level up or down by `GLOWSPEED` (8 units per tic) and reverses direction at the min/max bounds.

**Parameters:**
- `g` - Pointer to the glow thinker state.

**Key logic:**
- `direction == -1` (going down): subtracts `GLOWSPEED` from `sector->lightlevel`. When it would go below `minlight`, adds `GLOWSPEED` back and flips direction to +1.
- `direction == 1` (going up): adds `GLOWSPEED`. When it reaches or exceeds `maxlight`, subtracts `GLOWSPEED` and flips to -1.
- This creates a smooth, bouncing oscillation between the two levels.

---

### `P_SpawnGlowingLight`

```c
void P_SpawnGlowingLight(sector_t* sector)
```

**Purpose:** Initializes and registers a glowing light thinker during level load.

**Parameters:**
- `sector` - The sector to apply the glow effect to.

**Key logic:**
- `minlight = P_FindMinSurroundingLight(sector, sector->lightlevel)`.
- `maxlight = sector->lightlevel` (current level is the ceiling).
- Initial `direction = -1` (starts going down).
- Zeroes `sector->special`.

---

## Data Structures

The thinker structures used in this file are defined in `p_spec.h`:

### `fireflicker_t`

Used by `T_FireFlicker` / `P_SpawnFireFlicker`.

| Field | Type | Description |
|-------|------|-------------|
| `thinker` | `thinker_t` | Linked-list node and function pointer for the thinker system |
| `sector` | `sector_t*` | The sector being affected |
| `count` | `int` | Tic countdown until the next light update |
| `maxlight` | `int` | Upper light level (sector's original level) |
| `minlight` | `int` | Lower light level (min surrounding + 16) |

### `lightflash_t`

Used by `T_LightFlash` / `P_SpawnLightFlash`.

| Field | Type | Description |
|-------|------|-------------|
| `thinker` | `thinker_t` | Thinker linkage |
| `sector` | `sector_t*` | The affected sector |
| `count` | `int` | Tic countdown |
| `maxlight` | `int` | Bright phase level |
| `minlight` | `int` | Dark phase level |
| `maxtime` | `int` | Maximum random duration for bright phase (64) |
| `mintime` | `int` | Maximum random duration for dark phase (7) |

### `strobe_t`

Used by `T_StrobeFlash` / `P_SpawnStrobeFlash`.

| Field | Type | Description |
|-------|------|-------------|
| `thinker` | `thinker_t` | Thinker linkage |
| `sector` | `sector_t*` | The affected sector |
| `count` | `int` | Tic countdown |
| `minlight` | `int` | Dark phase level |
| `maxlight` | `int` | Bright phase level |
| `darktime` | `int` | Fixed duration of dark phase |
| `brighttime` | `int` | Fixed duration of bright phase (`STROBEBRIGHT`) |

### `glow_t`

Used by `T_Glow` / `P_SpawnGlowingLight`.

| Field | Type | Description |
|-------|------|-------------|
| `thinker` | `thinker_t` | Thinker linkage |
| `sector` | `sector_t*` | The affected sector |
| `minlight` | `int` | Lower oscillation bound |
| `maxlight` | `int` | Upper oscillation bound |
| `direction` | `int` | Current direction: `+1` up, `-1` down |

---

## Dependencies

| File | Why needed |
|------|-----------|
| `z_zone.h` | `Z_Malloc` for allocating thinker structures with `PU_LEVSPEC` tag |
| `m_random.h` | `P_Random()` for randomized amounts in fire flicker and light flash |
| `doomdef.h` | Core types and constants |
| `p_local.h` | `P_AddThinker`, `P_FindMinSurroundingLight`, `P_FindSectorFromLineTag`, sector helper functions |
| `r_state.h` | `sectors[]` array and `numsectors` count, accessed in `EV_TurnTagLightsOff` and `EV_LightTurnOn` |
