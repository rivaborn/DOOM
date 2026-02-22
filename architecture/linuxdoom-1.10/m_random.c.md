# File Overview

**File:** `linuxdoom-1.10/m_random.c`
**Module prefix:** `M_` (general) and `P_` (play simulation)

`m_random.c` implements DOOM's pseudorandom number generator (PRNG). Rather than using a mathematical formula at runtime, DOOM's PRNG is a **256-entry lookup table** (`rndtable[]`) that is indexed by a simple wrapping counter. Advancing the counter by one and reading the corresponding table entry yields the next "random" value in the sequence.

This design has one critical property: the sequence is **completely deterministic and reproducible** as long as the starting index is known. This determinism is the foundation of DOOM's **demo recording and playback system** - a demo file stores only the sequence of player inputs, not the resulting game state. On playback, as long as the PRNG is reset to index 0 and driven in exactly the same order as during recording, every monster movement, damage roll, and projectile spread will be identical.

Two separate indices are maintained - one for gameplay simulation (`prndindex`, used by `P_Random`) and one for non-gameplay effects (`rndindex`, used by `M_Random`) - so that visual-only randomness (particle effects, menu animations, etc.) does not desynchronize the gameplay sequence and break demo playback.

---

## Global Variables

### `rndtable[256]`

```c
unsigned char rndtable[256];
```

A fixed, hardcoded 256-byte lookup table containing pre-selected values in the range 0-255. This table was hand-chosen by id Software and is identical in every DOOM binary ever shipped. Its exact contents are part of the DOOM specification - changing any byte would break compatibility with all existing demo files and networked multiplayer sessions.

The table values are not uniformly distributed or cryptographically random; they were chosen to appear random for game purposes. The full sequence cycles after exactly 256 calls.

```
  0,   8, 109, 220, 222, 241, 149, 107,  75, 248, 254, 140,  16,  66,
 74,  21, 211,  47,  80, 242, 154,  27, 205, 128, 161,  89,  77,  36,
 95, 110,  85,  48, 212, 140, 211, 249,  22,  79, 200,  50,  28, 188,
 52, 140, 202, 120,  68, 145,  62,  70, 184, 190,  91, 197, 152, 224,
149, 104,  25, 178, 252, 182, 202, 182, 141, 197,   4,  81, 181, 242,
145,  42,  39, 227, 156, 198, 225, 193, 219,  93, 122, 175, 249,   0,
175, 143,  70, 239,  46, 246, 163,  53, 163, 109, 168, 135,   2, 235,
 25,  92,  20, 145, 138,  77,  69, 166,  78, 176, 173, 212, 166, 113,
 94, 161,  41,  50, 239,  49, 111, 164,  70,  60,   2,  37, 171,  75,
136, 156,  11,  56,  42, 146, 138, 229,  73, 146,  77,  61,  98, 196,
135, 106,  63, 197, 195,  86,  96, 203, 113, 101, 170, 247, 181, 113,
 80, 250, 108,   7, 255, 237, 129, 226,  79, 107, 112, 166, 103, 241,
 24, 223, 239, 120, 198,  58,  60,  82, 128,   3, 184,  66, 143, 224,
145, 224,  81, 206, 163,  45,  63,  90, 168, 114,  59,  33, 159,  95,
 28, 139, 123,  98, 125, 196,  15,  70, 194, 253,  54,  14, 109, 226,
 71,  17, 161,  93, 186,  87, 244, 138,  20,  52, 123, 251,  26,  36,
 17,  46,  52, 231, 232,  76,  31, 221,  84,  37, 216, 165, 212, 106,
197, 242,  98,  43,  39, 175, 254, 145, 190,  84, 118, 222, 187, 136,
120, 163, 236, 249
```

---

### `rndindex`

```c
int rndindex = 0;
```

Current position in `rndtable[]` for the **non-deterministic** stream, advanced by `M_Random`. Used for visual effects, menu randomness, and other things that must not affect demo sync.

---

### `prndindex`

```c
int prndindex = 0;
```

Current position in `rndtable[]` for the **deterministic gameplay** stream, advanced by `P_Random`. This index is the critical piece of shared state in networked multiplayer and demo playback. If two clients or a client and a demo differ in `prndindex` at any point, the game will immediately desynchronize.

---

## Functions

### `P_Random`

```c
int P_Random(void)
```

**Purpose:** Advance the gameplay RNG index by one and return the next value from `rndtable[]`. This is the **only** function that should be called for any game-logic randomness: monster AI decisions, damage calculations, projectile spread, item respawn, sound pitch variation for in-game sounds, and similar.

**Parameters:** None.

**Returns:** An integer in the range 0-255.

**Key logic:**
```c
prndindex = (prndindex + 1) & 0xff;   // Wrap at 256
return rndtable[prndindex];
```

The `& 0xff` mask replaces a modulo operation and ensures the index wraps cleanly from 255 back to 0, giving a repeating period of exactly 256. The sequence is identical on every platform and compiler because it is purely a table lookup with no floating-point or architecture-specific operations.

**Demo sync contract:** Every call to `P_Random` must occur in the same order on every machine running the same session. Any code path that calls `P_Random` under conditions that differ between machines (different frame rates, different platform random behaviour, etc.) will desynchronize the game. This is why rendering, menu code, and particle effects use `M_Random` instead.

---

### `M_Random`

```c
int M_Random(void)
```

**Purpose:** Advance the non-gameplay RNG index by one and return the next value from `rndtable[]`. Used for effects that must look random but have no effect on reproducible game simulation: screen wipe randomness, menu animations, and similar.

**Parameters:** None.

**Returns:** An integer in the range 0-255.

**Key logic:**
```c
rndindex = (rndindex + 1) & 0xff;
return rndtable[rndindex];
```

Structurally identical to `P_Random` but uses the separate `rndindex` counter. Advancing `rndindex` never affects `prndindex`, so calling `M_Random` any number of times cannot desynchronize a running demo or network game.

---

### `M_ClearRandom`

```c
void M_ClearRandom(void)
```

**Purpose:** Reset both RNG indices to zero, restoring the PRNG to its initial state. Called at the start of each new game or demo playback to guarantee that the random sequence starts from a known, reproducible position.

**Parameters:** None.

**Returns:** Nothing.

**Key logic:**
```c
rndindex = prndindex = 0;
```

After this call, the very first `P_Random()` call will advance `prndindex` to 1 and return `rndtable[1]` (value `8`). This is the defined starting state for all DOOM demos.

---

## Data Structures

None. The module uses only a plain `unsigned char` array and two `int` index variables.

---

## The Two-Stream Design and Demo Compatibility

DOOM's demo format stores player input tics and relies on the engine reproducing identical game state from those inputs. The PRNG is the key mechanism that makes this possible:

- **Recording:** `prndindex` starts at 0. As the game runs, every call to `P_Random` advances it deterministically. The demo file does not store the RNG state; it implicitly starts at 0.
- **Playback:** `M_ClearRandom` resets `prndindex` to 0 before replay begins. The game then executes the recorded inputs, and because `P_Random` advances in precisely the same sequence, all outcomes are identical.
- **Network play:** All peers start with `prndindex = 0` and advance it in lockstep via the netgame input synchronisation. A desync in `prndindex` causes visibly diverging monster behaviour ("out of sync" error).

The separation of `rndindex` (for `M_Random`) means that cosmetic randomness - which may vary between machines due to different rendering frame rates or UI interaction - does not contaminate the gameplay sequence.

---

## Dependencies

| Module | Header | What is used |
|--------|--------|-------------|
| DOOM types | `doomtype.h` | Included indirectly; no direct include in this `.c` file |

`m_random.c` has no `#include` directives of its own (beyond the implicit C runtime). It is a self-contained table and counter. The functions are declared in `m_random.h` which is included by callers.
