# File Overview

`m_cheat.h` is the header for DOOM's cheat code detection module. It defines the `SCRAMBLE` bit-permutation macro used to obfuscate cheat sequences, the `cheatseq_t` state structure used to track matching progress for each cheat, and the two public functions that check and extract cheat input.

The obfuscation scheme prevents cheat codes from appearing as readable ASCII strings in the compiled binary, making them slightly harder to discover via hex editor inspection.

## Global Variables

No global variables are declared in this header.

## Functions

### `cht_CheckCheat`

**Signature:** `int cht_CheckCheat(cheatseq_t* cht, char key)`

**Purpose:** Advances a cheat sequence state machine by one key press. Returns 1 when the complete cheat sequence has been successfully entered.

**Parameters:**
- `cht` - Pointer to the cheat state structure for the specific cheat being tested
- `key` - The key character just pressed by the player

**Return value:** 1 if the cheat is now complete; 0 otherwise.

---

### `cht_GetParam`

**Signature:** `void cht_GetParam(cheatseq_t* cht, char* buffer)`

**Purpose:** Extracts parameter characters typed as part of a cheat code into the provided buffer. Must be called immediately after `cht_CheckCheat` returns 1 for a parameterized cheat.

**Parameters:**
- `cht` - Pointer to the cheat state (contains the typed parameter bytes)
- `buffer` - Destination for the extracted parameter string (null-terminated)

**Return value:** None.

## Data Structures

### `SCRAMBLE` Macro

```c
#define SCRAMBLE(a) \
((((a)&1)<<7) + (((a)&2)<<5) + ((a)&4) + (((a)&8)<<1) \
 + (((a)&16)>>1) + ((a)&32) + (((a)&64)>>5) + (((a)&128)>>7))
```

A bit-permutation macro that scrambles/unscrambles a byte value by rearranging its bits. The permutation maps:
- bit 0 -> bit 7
- bit 1 -> bit 6
- bit 2 -> bit 2 (unchanged)
- bit 3 -> bit 4
- bit 4 -> bit 3
- bit 5 -> bit 5 (unchanged)
- bit 6 -> bit 1
- bit 7 -> bit 0

The permutation is its own inverse: `SCRAMBLE(SCRAMBLE(x)) == x`. This property means the same macro is used for both encoding the stored cheat sequences and encoding incoming keypresses for comparison — no separate decode function is needed.

---

### `cheatseq_t`

```c
typedef struct
{
    unsigned char*  sequence;  // Pointer to the scrambled cheat byte sequence
    unsigned char*  p;         // Current position pointer within sequence
} cheatseq_t;
```

**Fields:**
- `sequence`: Points to a byte array containing the scrambled cheat key sequence followed by sentinel bytes. The sentinel values are:
  - `0`: Parameter placeholder — when the state machine reaches a `0` byte, subsequent keys are stored here as typed parameters (for cheats like `IDMUS`, `IDCLEV`)
  - `1`: Parameter start marker — separates the fixed cheat prefix from the parameter section
  - `0xff`: End of sequence — signals successful completion of the cheat

- `p`: Tracks the current matching position within `sequence`. NULL before first use (auto-initialized to `sequence` by `cht_CheckCheat`). On a mismatch, reset to `sequence`. On completion, reset to `sequence` and return 1.

**Usage pattern in `st_stuff.c`:**
```c
// Static definition of a cheat (stored in scrambled form):
cheatseq_t cheat_iddqd = { (unsigned char*) "\xb2\xb6\xcd\xcb\xc3", 0 };

// Per-key check in the status bar responder:
if (cht_CheckCheat(&cheat_iddqd, ev->data1))
{
    // Cheat activated - grant god mode
    player->cheats ^= CF_GODMODE;
}
```

## Dependencies

No headers are included in `m_cheat.h` itself. The types `unsigned char` and `char` are fundamental C types.
