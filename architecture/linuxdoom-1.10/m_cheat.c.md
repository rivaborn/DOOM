# File Overview

`m_cheat.c` implements DOOM's cheat code detection system. It provides a state-machine-based approach to recognizing typed cheat sequences (like IDDQD, IDKFA, IDCLIP, etc.) as the player presses keys during gameplay.

The system works by maintaining a `cheatseq_t` structure for each cheat code. Each structure contains the scrambled sequence of bytes to match against and a pointer into that sequence tracking progress. As each key is pressed, it is compared against the expected next byte in the sequence. A successful match advances the pointer; a mismatch resets it to the beginning.

**Scrambling:** The cheat sequences are stored in a scrambled/encoded form to prevent them from appearing as readable strings in the binary. The `SCRAMBLE` macro performs a bit-permutation that rearranges the bits of each byte. The same macro is used to encode the expected sequences (in `st_stuff.c`) and to encode each incoming key before comparison. This obfuscation is noted as basic — it was enough to prevent casual inspection of the binary with a hex editor in the 1990s.

**Parameter cheats:** Some cheats accept parameters typed after the base sequence (e.g., `IDMUS` followed by a two-digit music number, or `IDCLEV` followed by a level number). These use a special byte value `0` embedded in the sequence to mark where parameter collection begins, and `0xff` to mark the end of the sequence.

**Called from:** `st_stuff.c` (the status bar module), which processes keyboard input during gameplay and calls `cht_CheckCheat` for each registered cheat.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `static int` | `firsttime` | Initialization guard flag. Set to 1 initially; cleared to 0 on the first call to `cht_CheckCheat`. Triggers one-time initialization of `cheat_xlate_table`. |
| `static unsigned char` | `cheat_xlate_table[256]` | The scrambled translation table. Maps every possible input byte value to its scrambled equivalent via the `SCRAMBLE` macro. Initialized lazily on first use. Pre-computing this table avoids re-computing the macro for every key comparison. |

## Functions

### `cht_CheckCheat`

**Signature:** `int cht_CheckCheat(cheatseq_t* cht, char key)`

**Purpose:** Advances the state machine for a given cheat sequence by one character. Returns 1 if the full cheat sequence has just been completed, 0 otherwise.

**Parameters:**
- `cht` - Pointer to the `cheatseq_t` state structure for the cheat being checked. Contains the expected sequence and a pointer to the current matching position.
- `key` - The most recently pressed key character to test against the cheat sequence.

**Return value:**
- `1` if the cheat code was successfully completed (the `0xff` end marker was reached)
- `0` if the cheat is not yet complete or the sequence was broken

**Key logic:**
1. **One-time initialization**: On `firsttime`, fills `cheat_xlate_table[i] = SCRAMBLE(i)` for all 256 byte values.
2. **Pointer initialization**: If `cht->p` is NULL, sets it to `cht->sequence` (start of the sequence).
3. **Parameter collection mode**: If `*cht->p == 0` (the parameter marker byte), stores the raw `key` directly into `*(cht->p++)` without scramble comparison. This collects user-typed characters into the sequence buffer.
4. **Normal matching mode**: If `cheat_xlate_table[(unsigned char)key] == *cht->p`, the scrambled key matches the expected scrambled byte — advance `cht->p`.
5. **Mismatch**: If neither condition matched, reset `cht->p = cht->sequence` (start over).
6. **Skip byte 1**: If `*cht->p == 1`, advance the pointer past it (sentinel byte).
7. **End of sequence**: If `*cht->p == 0xff`, the full sequence has been matched. Reset `cht->p = cht->sequence` and return `rc = 1`.
8. Returns `rc` (0 unless step 7 occurred).

---

### `cht_GetParam`

**Signature:** `void cht_GetParam(cheatseq_t* cht, char* buffer)`

**Purpose:** Extracts the parameter characters that were typed as part of a parameter-accepting cheat code (e.g., extracts "05" from `IDMUS05`). Called after `cht_CheckCheat` returns 1, before resetting the cheat state.

**Parameters:**
- `cht` - Pointer to the `cheatseq_t` state structure. The sequence buffer has been modified in-place to store the typed parameter characters.
- `buffer` - Destination buffer for the extracted parameter string. Caller must ensure adequate size.

**Return value:** None.

**Key logic:**
1. Scans `cht->sequence` forward until it finds the byte value `1` (the parameter-start sentinel). This `while (*(p++) != 1)` loop positions `p` just after the `1` byte.
2. Copies characters from the sequence into `buffer`, simultaneously zeroing each source byte (`*(p++) = 0`) to reset the sequence for future use.
3. Stops when it encounters a null byte `c == 0` (end of typed input) or when `*p == 0xff` (end-of-sequence marker).
4. If the loop ended at `0xff`, appends a null terminator to `buffer`.

The combined effect: extracts the typed parameter, clears it from the sequence buffer (so the cheat can be typed again), and null-terminates the output.

## Data Structures

### `cheatseq_t` (defined in `m_cheat.h`)

The `cheatseq_t` structure is the state machine representation for a single cheat code. It is defined in the header but used extensively here:

```c
typedef struct
{
    unsigned char*  sequence;  // The scrambled cheat sequence bytes (with sentinels)
    unsigned char*  p;         // Current matching position within sequence
} cheatseq_t;
```

- `sequence`: Points to a static array of bytes encoding the expected key sequence in scrambled form. Special sentinel values: `0` = parameter placeholder (filled in by `cht_CheckCheat`), `1` = parameter-start marker, `0xff` = end of sequence.
- `p`: Tracks progress through the sequence. NULL means uninitialized (will be set to `sequence` on first use). Advances on each correct key press; resets to `sequence` on mismatch or completion.

## Dependencies

| File | Reason |
|------|--------|
| `m_cheat.h` | Own header: defines `cheatseq_t`, `SCRAMBLE` macro, and function declarations |
