# File Overview

`m_argv.c` provides DOOM's command-line argument parsing system. It is intentionally minimal, offering only a single function to check for the presence of a named argument in the program's command line.

The module stores the argument count and argument vector as global variables (`myargc`, `myargv`), which are set at program startup from the `main` function's parameters and then made available throughout the entire engine via `m_argv.h`. This global accessibility allows any module to check for command-line flags without needing to pass `argc`/`argv` through the call stack.

The search is case-insensitive (using `strcasecmp`), so `-DEVPARM`, `-devparm`, and `-Devparm` are all equivalent.

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `int` | `myargc` | The number of command-line arguments, directly mirroring the `argc` parameter from `main`. Set at program startup in `i_main.c`. |
| `char**` | `myargv` | The array of command-line argument strings, directly mirroring the `argv` parameter from `main`. Set at program startup in `i_main.c`. `myargv[0]` is the program name; `myargv[1]` through `myargv[myargc-1]` are the actual arguments. |

## Functions

### `M_CheckParm`

**Signature:** `int M_CheckParm(char *check)`

**Purpose:** Searches the command-line argument list for a specific parameter string. Returns the argument's position (1-based index) if found, or 0 if not present. This is the primary interface used throughout the DOOM codebase to detect and locate command-line flags.

**Parameters:**
- `check` - The argument string to search for (e.g., `"-devparm"`, `"-warp"`, `"-skill"`). Comparison is case-insensitive.

**Return value:**
- Returns the 1-based index `i` (1 to `myargc-1`) of the matching argument in `myargv`.
- Returns `0` if the argument is not found.

The returned index is useful not only for detecting the flag's presence but also for locating any arguments that follow it (e.g., after finding `-warp` at position `i`, the episode and map numbers are at `myargv[i+1]` and `myargv[i+2]`).

**Key logic:**
1. Iterates from index 1 (skipping `myargv[0]`, the program name) to `myargc-1`.
2. For each argument, performs a case-insensitive comparison with `check` using `strcasecmp`.
3. Returns the first matching index.
4. Returns 0 if the loop completes without a match.

**Usage pattern throughout the engine:**
```c
// Simple presence check:
if (M_CheckParm("-devparm")) devparm = true;

// With following value:
p = M_CheckParm("-skill");
if (p && p < myargc-1)
    startskill = myargv[p+1][0] - '1';

// Guard against out-of-bounds:
if ((p=M_CheckParm("-warp")) && p < myargc-2)
    startmap = atoi(myargv[p+2]);
```

## Data Structures

No data structures are defined in this file.

## Dependencies

| File | Reason |
|------|--------|
| `<string.h>` | `strcasecmp` for case-insensitive string comparison |
