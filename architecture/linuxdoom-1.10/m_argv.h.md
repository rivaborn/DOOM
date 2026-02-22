# File Overview

`m_argv.h` is the header for DOOM's command-line argument access module. It exposes the two global variables that hold the raw command-line arguments and declares `M_CheckParm`, the sole function for querying those arguments.

This header is included throughout the DOOM codebase wherever command-line options need to be checked. It is one of the most widely included headers in the engine, as command-line flags control nearly every aspect of DOOM's startup behavior (WAD files to load, network settings, video mode, debugging options, demo recording/playback, etc.).

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `extern int` | `myargc` | The total count of command-line arguments passed to the program (equivalent to `argc` from `main`). Includes the program name at index 0. |
| `extern char**` | `myargv` | The array of null-terminated command-line argument strings (equivalent to `argv` from `main`). `myargv[0]` is the program name; subsequent entries are the user-supplied arguments. |

These are declared `extern` so that any translation unit including this header can access them. The actual storage is defined in `m_argv.c`, and the values are set in `i_main.c` (the platform-specific entry point that contains `main`).

## Functions

### `M_CheckParm`

**Signature:** `int M_CheckParm(char* check)`

**Purpose:** Case-insensitively searches the program's command-line argument list for the specified string. Returns the argument's index if found, or 0 if absent.

**Parameters:**
- `check` - Null-terminated string to search for (e.g., `"-devparm"`, `"-2"`, `"-warp"`)

**Return value:**
- The 1-based position of the argument in `myargv[]` if found (range: 1 to `myargc-1`)
- `0` if the argument is not present

The non-zero return value is useful both as a boolean (found/not-found) and as a position from which to read subsequent arguments.

## Data Structures

No data structures are defined in this header.

## Dependencies

No headers are included in this file. The `int` and `char` types are fundamental C types requiring no includes.
