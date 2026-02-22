# File Overview

`p_setup.h` is the minimal public interface header for the level setup module (`p_setup.c`). It exposes two functions: `P_SetupLevel` (called once per map transition) and `P_Init` (called once at startup). All of the map data arrays loaded by `p_setup.c` (`vertexes`, `sectors`, `lines`, etc.) are declared in `r_state.h`, not here, because they are also needed by the renderer.

## Functions (Declarations Only)

### `P_SetupLevel`
```c
void P_SetupLevel(int episode, int map, int playermask, skill_t skill);
```
**Purpose:** Loads a complete map and initializes all play and render subsystems for it.

**Parameters:**
- `episode` - Episode number (1–4 for DOOM, ignored for DOOM II).
- `map` - Map number within the episode.
- `playermask` - Bitmask indicating which player slots are active.
- `skill` - Skill level controlling monster spawning.

The comment `// NOT called by W_Ticker. Fixme.` is a historical note indicating this function was originally expected to be called from a ticker but was later called from the game state machine directly.

### `P_Init`
```c
void P_Init(void);
```
**Purpose:** One-time startup initialization. Initializes switch lists, picture animations, and sprite definitions. Must be called after WAD data is loaded.

## Dependencies

| File | Reason |
|------|--------|
| `doomdef.h` (implicit) | `skill_t` type definition |
