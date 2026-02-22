# File Overview

**File:** `linuxdoom-1.10/m_random.h`
**Module prefix:** `M_` / `P_`

This header is the public interface for `m_random.c`. It exposes three functions that together form DOOM's deterministic pseudorandom number generator (PRNG). Any module that needs random numbers must include this header and choose the correct function: `P_Random` for game-simulation code that must be reproducible across demo playback and network sessions, or `M_Random` for purely cosmetic randomness that does not affect game state.

The `P_Random` / `M_Random` split is one of the most important architectural decisions in DOOM - getting it wrong (calling `P_Random` from rendering code, or `M_Random` from gameplay code) will silently break demo compatibility or network synchronisation.

---

## Global Variables

None declared in this header. The `rndtable[]` array and the two index variables (`rndindex`, `prndindex`) are defined in `m_random.c` and are not exposed publicly - they should only be accessed through the functions below.

---

## Functions

### `M_Random`

```c
int M_Random(void);
```

**Purpose:** Return the next value (0-255) from the shared `rndtable[]` lookup table, advancing the **non-gameplay** index `rndindex`. Safe to call from rendering, menu, and visual-effect code without risking demo desynchronization.

**Parameters:** None.

**Returns:** An integer in the range `[0, 255]`.

**When to use:** Any randomness that is purely cosmetic and has no bearing on reproducible game simulation. Examples include screen-wipe effects, menu selection jitter, or particle spawn patterns that exist only on the local machine.

---

### `P_Random`

```c
int P_Random(void);
```

**Purpose:** Return the next value (0-255) from the shared `rndtable[]` lookup table, advancing the **gameplay** index `prndindex`. This is the authoritative RNG for all game-simulation code.

**Parameters:** None.

**Returns:** An integer in the range `[0, 255]`.

**When to use:** All randomness that is part of the reproducible game simulation and must be identical across demo recording/playback and all network peers. Mandatory for:
- Monster AI decisions (attack choices, movement angles, wake-up thresholds)
- Melee and projectile damage rolls
- Projectile spread (e.g., shotgun pellet angles)
- Item and monster respawn logic
- Sound pitch variation for in-game sounds

**Critical constraint:** Every call to `P_Random` on every machine in a session must occur in the same logical order. Any conditional call (one that happens on some machines but not others) will advance `prndindex` differently and desynchronize the game from that point forward.

---

### `M_ClearRandom`

```c
void M_ClearRandom(void);
```

**Purpose:** Reset both RNG indices (`rndindex` and `prndindex`) to zero, restoring the PRNG to its defined initial state. Must be called at the start of every new game level and before beginning demo playback to ensure the sequence starts from the same known position on all machines.

**Parameters:** None.

**Returns:** Nothing.

**Effect:** After this call, the first `P_Random()` returns `rndtable[1]` = `8`, and the first `M_Random()` returns `rndtable[1]` = `8` (since both start at index 0 and both advance before reading).

---

## Data Structures

None defined in this header.

---

## Dependencies

| Header | Purpose |
|--------|---------|
| `doomtype.h` | Provides `boolean` and other base types (included for consistency with other `m_*.h` headers, though the functions here only use `int`) |
