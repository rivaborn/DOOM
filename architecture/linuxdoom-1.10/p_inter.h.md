# File Overview

`p_inter.h` is the minimal public interface header for `p_inter.c`, the interaction handling module of the DOOM engine. It uses the GCC `#pragma interface` / `#pragma implementation` pair to mark this header as the "interface" counterpart to `p_inter.c`'s `#pragma implementation`.

The header intentionally exposes only a single function declaration: `P_GivePower`. All other functions defined in `p_inter.c` (`P_TouchSpecialThing`, `P_DamageMobj`, `P_GiveAmmo`, `P_GiveWeapon`, etc.) are declared instead in `p_local.h`, the master local-play header, which is included by all play subsystem files. This design reflects id Software's convention of using `p_local.h` as the central include for intra-subsystem communication and reserving per-file headers for the single function that needs a truly narrow public declaration.

---

## Global Variables

None declared.

---

## Functions

### `P_GivePower` (declaration only)

```c
boolean P_GivePower(player_t*, int);
```

**Purpose:** Activates a powerup for the given player. This is declared here (and implemented in `p_inter.c`) because it is called from code outside the play subsystem that does not include `p_local.h` directly.

**Parameters:**
- `player_t*` - Pointer to the player who receives the powerup.
- `int` - The powerup type, which is formally a `powertype_t` enum value but is cast to `int` in the declaration to reduce header coupling.

**Return value:** `boolean` - `true` if the power was successfully applied, `false` if the player already possesses it (for non-timed powers).

---

## Data Structures

None defined.

---

## Dependencies

| File | Why needed |
|------|-----------|
| (implicit) `doomtype.h` / `doomdef.h` | Provides `boolean` and `player_t` types used in the function signature; these must be in scope wherever this header is included |
