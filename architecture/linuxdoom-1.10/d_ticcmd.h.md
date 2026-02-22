# File Overview

`d_ticcmd.h` defines `ticcmd_t`, the single most important data structure for multiplayer synchronization and demo recording in DOOM. It captures all player input for one game tick (1/35th of a second) in a compact, transmissible format. Every networked game works by collecting `ticcmd_t` values from all players, broadcasting them to all peers, and then having every machine run the simulation with the same set of commands — ensuring a perfectly deterministic, synchronized game state across all nodes.

This header is intentionally minimal and isolated so it can be included anywhere without dragging in heavy dependencies.

## Global Variables

This header defines no global variables.

## Functions

This header declares no functions.

## Data Structures

### `ticcmd_t` (struct)

The per-tick player input command. Six bytes total, compact for efficient network transmission.

```c
typedef struct {
    char   forwardmove;   // Forward/backward movement delta. Scaled by 2048 for actual movement.
    char   sidemove;      // Strafe left/right movement delta. Scaled by 2048.
    short  angleturn;     // Horizontal turn delta. Left-shifted by 16 to get a BAM angle delta.
    short  consistancy;   // Network consistency check value (player X position or rndindex).
    byte   chatchar;      // In-game chat character (0 = none).
    byte   buttons;       // Bitfield of action buttons pressed this tick.
} ticcmd_t;
```

**Field details:**

- **`forwardmove`**: Signed byte. Positive = forward, negative = backward. The values `0x19` (slow) and `0x32` (fast/running) are the standard movement speeds defined in `g_game.c`. The actual world-space velocity is this value multiplied by `FRACUNIT*2` in `p_user.c`.

- **`sidemove`**: Signed byte. Positive = strafe right, negative = strafe left. Standard values are `0x18` (slow) and `0x28` (fast).

- **`angleturn`**: Signed short. Added to the player's angle (BAM = Binary Angle Measurement format). In `G_BuildTiccmd`, mouse X-movement and keyboard/joystick turns are accumulated here. The value is left-shifted by 16 bits when applied to the player's angle to convert from this compact form to full 32-bit BAM.

- **`consistancy`**: Used only in multiplayer. Each machine stores either `players[i].mo->x` or `rndindex` (random number index) in this field when building a command, then checks on receipt that the other machine's value matches its own — detecting desync. A mismatch causes `I_Error("consistency failure")`.

- **`chatchar`**: The ASCII character typed in chat this tick, or 0. In-game chat is built character-by-character via `HU_dequeueChatChar()`.

- **`buttons`**: A packed bitfield controlling actions. The bit definitions come from `g_game.h`:
  - `BT_ATTACK` — Fire button.
  - `BT_USE` — Use/interact button.
  - `BT_CHANGE` — Weapon change flag (active when a weapon number is pressed).
  - `BT_WEAPONSHIFT` / `BT_WEAPONMASK` — Encodes the selected weapon number within the buttons byte.
  - `BT_SPECIAL` — Special command flag.
  - `BT_SPECIALMASK` — Mask for the special command type:
    - `BTS_PAUSE` — Toggle pause.
    - `BTS_SAVEGAME` — Trigger a save game.
    - `BTS_SAVEMASK` / `BTS_SAVESHIFT` — Encodes the save slot within the special command.

**Demo format:** When recording demos, `G_WriteDemoTiccmd` serializes only four bytes per tic (`forwardmove`, `sidemove`, `angleturn` high byte, `buttons`), omitting `consistancy` and `chatchar` to save space. Playback reads these same four bytes.

## Dependencies

| File | Reason |
|------|--------|
| `doomtype.h` | Provides `byte` (`unsigned char`) and `boolean` typedefs used in this and related headers. |
