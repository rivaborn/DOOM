# File Overview

**Path:** `sndserv/soundsrv.c`
**RCS:** `$Id: soundsrv.c,v 1.3 1997/01/29 22:40:44 b1 Exp $`

`soundsrv.c` is the main entry point and core logic of the DOOM sound server: a standalone Unix process launched as a child by the main DOOM executable to handle all PCM sound-effect mixing and output. It was originally conceived for SGI IRIX and is most prominently used with the Linux OSS/Voxware sound driver.

## Role in the DOOM Engine

DOOM's Linux port separates sound output into a dedicated child process to avoid the complexity of mixing audio inside the game loop while also managing rendering and input. The main DOOM process (`linuxdoom-1.10`) communicates with this server exclusively through **stdin/stdout pipes** (IPC via file descriptors 0 and 1). The sound server:

1. Reads a WAD file to preload all sound-effect sample data.
2. Enters a main loop, polling `stdin` with `select()` for play/quit/save commands from the parent.
3. Mixes up to 8 simultaneous audio channels using a software pitch-shifting and stereo-panning mixer.
4. Submits mixed PCM buffers to the audio hardware via platform-specific `I_SubmitOutputBuffer()`.

## IPC Mechanism

Communication between the DOOM parent process and the sound server is entirely text-based, over a pair of Unix pipes connected to the sound server's `stdin` (fd 0) and `stdout` (fd 1).

### Commands received on stdin

Each command is a single ASCII character followed by additional hex-encoded bytes (no whitespace separators other than a trailing newline on responses):

| Command byte | Total bytes read after cmd | Format | Meaning |
|---|---|---|---|
| `'p'` | 8 more bytes + implicit newline | `p<snd#><step><vol><sep>` | Play a sound effect. Each field is 2 hex nibbles (1 byte each) encoding an 8-bit value. `snd#` = sound effect index, `step` = pitch step table index (0–255), `vol` = volume (0–127), `sep` = stereo separation (0–255). |
| `'q'` | 1 more byte (ignored) | `q<x>` | Quit-when-finished: stop accepting new sounds, wait for all active channels to finish, then exit. |
| `'s'` | 3 more bytes | `s<nn><filename>` | Save sound: dump raw PCM bytes for sound `nn` to a file whose name is given in the 2 remaining bytes (treated as a null-terminated string after zeroing byte 2). Debug/utility command. |

### Responses written to stdout

The function `outputushort()` writes a 4-nibble hex string followed by `'\n'` (5 bytes total) to fd 1 to report the channel handle assigned to a newly started sound. In the released code the actual call to `outputushort()` inside the `'p'` handler is commented out, so no response is sent in practice.

End-of-file on stdin (i.e., the parent process closed the pipe or exited) is detected when `read()` returns 0, which sets `done = 1` and terminates the loop.

---

## Global Variables

| Type | Name | Purpose |
|---|---|---|
| `static int` | `mytime` | Monotonically incrementing tick counter incremented once per main-loop iteration. Used as a timestamp to determine which channel started earliest (for eviction). |
| `int` | `numsounds` | Number of sound effects; set to `NUMSFX` in `grabdata()`. |
| `int` | `longsound` | Length in samples of the longest loaded sound effect. |
| `int[NUMSFX]` | `lengths` | Per-sound-effect byte lengths (post-padding, excluding the 8-byte WAD header). Indexed by `sfxenum_t`. |
| `signed short[MIXBUFFERSIZE]` | `mixbuffer` | Stereo interleaved 16-bit mixing output buffer. Size is `SAMPLECOUNT * 2 * 2` (512 samples * 2 channels * 2 bytes). |
| `int` | `sfxdevice` | File descriptor for the sound effects audio device (set in `linux.c`). |
| `int` | `musdevice` | File descriptor for the music device (set in `linux.c`). |
| `unsigned char*[8]` | `channels` | Per-channel read pointers into the raw sample data for the currently playing sound. `NULL` means the channel is inactive. |
| `unsigned int[8]` | `channelstep` | Per-channel fixed-point pitch step value (16.16 format). Derived from `steptable[]`. |
| `unsigned int[8]` | `channelstepremainder` | Per-channel fractional accumulator for sub-sample pitch stepping (the 0.16 fractional part). |
| `unsigned char*[8]` | `channelsend` | Per-channel pointer to one byte past the last valid sample byte. When `channels[i] >= channelsend[i]` the channel is silenced. |
| `int[8]` | `channelstart` | The value of `mytime` when each channel started, used for LRU eviction when all 8 channels are occupied. |
| `int[8]` | `channelhandles` | Monotonically increasing 16-bit handle values assigned to each active channel, returned to the parent as the channel identifier. |
| `int*[8]` | `channelleftvol_lookup` | Per-channel pointer into `vol_lookup[]` for left-channel volume scaling. Points to the row corresponding to the channel's computed left volume. |
| `int*[8]` | `channelrightvol_lookup` | Per-channel pointer into `vol_lookup[]` for right-channel volume scaling. Points to the row corresponding to the channel's computed right volume. |
| `int[8]` | `channelids` | Sound-effect ID (`sfxenum_t`) of the sound currently occupying each channel. Used to enforce singularity for certain effects. |
| `int` | `snd_verbose` | Verbosity flag. `1` by default; set to `0` by `-quiet` command-line argument. Controls diagnostic output to `stderr`. |
| `int[256]` | `steptable` | Pitch-step lookup table. Entry `i` maps to a 16.16 fixed-point resampling step. Entry 128 = `1.0` (native rate). Built by `initdata()` as `pow(2, (i-128)/64) * 65536`. |
| `int[128*256]` | `vol_lookup` | Volume/sample-conversion lookup table. `vol_lookup[vol*256 + sample]` converts an unsigned 8-bit sample to a scaled signed 16-bit contribution. Rows are volume levels 0–127; columns are sample values 0–255. Values are `(vol * (sample - 128) * 256) / 127`. |
| `static struct timeval` | `last` | Timestamp of the last timing reference. Initialized in `initdata()` but otherwise unused in the mixing path (legacy field). |
| `static struct timezone` | `whocares` | Timezone argument passed to `gettimeofday()`; not used. |
| `fd_set` | `fdset` | Permanent `select()` fd set containing only fd 0 (stdin). |
| `fd_set` | `scratchset` | Copy of `fdset` passed to `select()` each iteration (required because `select()` modifies the set in place). |

---

## Functions

### `static void derror(char* msg)`

**Purpose:** Fatal error handler. Prints `"error: <msg>"` to `stderr` and calls `exit(-1)`.

**Parameters:**
- `msg` - Human-readable error description string.

**Returns:** Never returns.

---

### `int mix(void)`

**Purpose:** Core software audio mixer. Iterates over `SAMPLECOUNT` (512) output stereo sample pairs, accumulates contributions from all 8 active channels into left (`dl`) and right (`dr`) accumulators, clamps each to the 16-bit signed range `[-0x8000, 0x7fff]`, and writes the result into `mixbuffer` as interleaved stereo 16-bit PCM.

**Parameters:** None.

**Returns:** Always `1`.

**Key Logic:**

- `leftout` and `rightout` walk through `mixbuffer` in steps of 2 (interleaved stereo).
- For each active channel (non-NULL `channels[i]`):
  - Reads the current sample byte (unsigned 8-bit).
  - Looks up the pre-scaled signed value from `channelleftvol_lookup[i][sample]` and `channelrightvol_lookup[i][sample]` and adds to `dl`/`dr`.
  - Advances the channel pointer using fixed-point arithmetic: `channelstepremainder[i] += channelstep[i]`; pointer advances by `channelstepremainder[i] >> 16`; remainder masked back to 16 bits. This implements nearest-neighbour sample-rate conversion.
  - If the advanced pointer reaches or passes `channelsend[i]`, the channel pointer is set to `NULL` (channel goes silent naturally).
- Output samples are clamped to `[-0x8000, 0x7fff]` (16-bit saturation), replacing an earlier 8-bit clamp.
- The body for each of the 8 channels is copy-pasted (unrolled by hand) rather than using a loop, which was a common optimization technique at the time.

---

### `void grabdata(int c, char** v)`

**Purpose:** Initialization phase - locates a suitable WAD file, opens it, and preloads all sound-effect sample data into `S_sfx[].data`.

**Parameters:**
- `c` - `argc` from `main()`.
- `v` - `argv` from `main()`.

**Returns:** void.

**Key Logic:**

1. Reads the `DOOMWADDIR` environment variable (defaults to `"."` if not set) to determine where to look for WAD files.
2. Builds full paths for: `doom2f.wad`, `doom2.wad`, `doomu.wad`, `doom.wad`, `doom1.wad` in priority order.
3. Scans `argv` for `-quiet` to suppress verbose output.
4. Uses `access()` to find the first readable WAD, then calls `openwad()` to open and index it.
5. Iterates over all `NUMSFX` sound effects:
   - If `S_sfx[i].link` is NULL: calls `getsfx(S_sfx[i].name, &lengths[i])` to load the raw PCM from the WAD lump named `"ds" + sfx_name`, storing the pointer in `S_sfx[i].data`. Tracks `longsound` (the maximum length seen).
   - If `S_sfx[i].link` is non-NULL: the sound is an alias - copies the data pointer and length from the linked entry. The length index computation `(S_sfx[i].link - S_sfx)/sizeof(sfxinfo_t)` is technically wrong (divides byte offset by struct size rather than using array-element difference), but works in practice when the link always points to an earlier element.

---

### `void updatesounds(void)`

**Purpose:** Single tick of audio output: calls `mix()` to fill `mixbuffer`, then calls `I_SubmitOutputBuffer()` to send the mixed data to the audio device.

**Parameters:** None.

**Returns:** void.

---

### `int addsfx(int sfxid, int volume, int step, int seperation)`

**Purpose:** Starts playing a new sound effect by allocating a channel, setting up all per-channel parameters (data pointer, step, volume lookups, handle), and returning the channel handle.

**Parameters:**
- `sfxid` - Index into `S_sfx[]` / `sfxenum_t` value.
- `volume` - Volume level 0–127.
- `step` - Pre-looked-up 16.16 fixed-point pitch step (already indexed through `steptable[]`).
- `seperation` - Stereo separation value 0–255; 0 = full left, 128 = center, 255 = full right.

**Returns:** The channel handle (a monotonically increasing 16-bit integer starting at 100), or `-1` (the initial value of `rc`, though `rc` is always overwritten before return).

**Key Logic:**

1. **Singularity enforcement:** For the sounds `sfx_sawup`, `sfx_sawidl`, `sfx_sawful`, `sfx_sawhit`, `sfx_stnmov`, `sfx_pistol`, any channel already playing the same `sfxid` is immediately stopped (pointer set to NULL) before allocating a new channel.
2. **Channel allocation:** Scans channels 0–7 for the first NULL (free) slot. If all 8 are occupied, the slot with the smallest `channelstart` value (oldest sound) is evicted.
3. **Setup:** Sets `channels[slot]` to the sound data pointer, `channelsend[slot]` to end of data, assigns handle (from `handlenums++`, wrapping-safe), stores step, clears remainder, records start time.
4. **Volume / pan calculation:**
   - Separation is shifted to the range 1–256.
   - Left volume: `volume - (volume * sep * sep) / (256*256)` — falls off quadratically with separation from left.
   - Right volume: same formula but `sep` is shifted by `-257` making it symmetrically negative, so right volume falls off as the sound moves left.
   - Both volumes are bounds-checked (0–127); out-of-range triggers `derror()`.
   - `channelleftvol_lookup[slot]` and `channelrightvol_lookup[slot]` are set to pointers into `vol_lookup[]` at the appropriate volume row.

---

### `void outputushort(int num)`

**Purpose:** Writes a 4-digit lowercase hexadecimal string followed by `'\n'` (5 bytes total) to `stdout` (fd 1). If `num < 0`, writes the literal string `"xxxx\n"` instead.

**Parameters:**
- `num` - A 16-bit unsigned value to encode, or `-1` to signal an error/invalid handle.

**Returns:** void.

**Key Logic:** Manually extracts four 4-bit nibbles from `num` and converts each to an ASCII hex digit (`'0'`–`'9'`, `'a'`–`'f'`). Uses `write(1, ...)` directly rather than `printf` to avoid buffering. Note: in the released code the call-site in the `'p'` command handler is commented out.

---

### `void initdata(void)`

**Purpose:** Initializes all channel state to idle, timestamps `last` via `gettimeofday()`, builds the `steptable[]` pitch lookup, and builds the `vol_lookup[]` volume/sample-conversion table.

**Parameters:** None.

**Returns:** void.

**Key Logic:**

- Zeroes all `channels[]` pointers.
- **steptable:** Computed as `pow(2.0, (i/64.0)) * 65536.0` for `i` in `[-128, 128)`, centered at index 128. This gives exact doubling per octave; index 128 = native rate (65536 = `1.0` in 16.16), index 64 = half rate (one octave down), index 192 = double rate (one octave up).
- **vol_lookup:** `vol_lookup[v*256 + s] = (v * (s - 128) * 256) / 127`. Converts unsigned 8-bit samples (0–255) to a signed 16-bit-scaled value, with `128` as the silence midpoint. Multiplying by 256 scales into 16-bit territory before mixing across 8 channels.

---

### `void quit(void)`

**Purpose:** Clean shutdown. Calls `I_ShutdownMusic()` and `I_ShutdownSound()` (platform-specific teardown in `linux.c`), then `exit(0)`.

**Parameters:** None.

**Returns:** Never returns.

---

### `int main(int c, char** v)`

**Purpose:** Program entry point. Orchestrates initialization, then runs the main event loop: polling stdin for IPC commands and calling `updatesounds()` every iteration.

**Parameters:**
- `c` - `argc`.
- `v` - `argv`.

**Returns:** `0` (though in practice `quit()` calls `exit(0)` before the `return` is reached).

**Key Logic:**

1. Calls `grabdata()`, `initdata()`, `I_InitSound(11025, 16)`, `I_InitMusic()`.
2. Initializes `fdset` to watch only fd 0 (stdin).
3. **Main loop** (`while (!done)`):
   a. Increments `mytime`.
   b. If not in `waitingtofinish` mode, calls `select()` with `zerowait` (non-blocking poll). While `select()` reports data available:
      - Reads 1 byte into `commandbuf[0]`.
      - EOF (`nrc == 0`) sets `done = 1`.
      - Dispatches on command character: `'p'` (play), `'q'` (quit-when-done), `'s'` (save).
      - **`'p'` decoding:** Reads 8 more bytes; each byte is an ASCII hex nibble, decoded by subtracting `'0'` or `'a'-10`. Reconstructed into 4 pairs: `sndnum`, raw `step` index, `vol`, `sep`. `step` is passed through `steptable[]` before calling `addsfx()`.
      - **`'q'` handling:** Sets `waitingtofinish = 1`, breaks out of inner loop.
      - **`'s'` handling:** Reads 3 more bytes: 2 hex nibbles for sound number, then treats the buffer as a filename (null-terminated by zeroing byte 2), opens the file, and writes the raw PCM data.
      - `select()` returning negative calls `quit()`.
   c. Calls `updatesounds()` unconditionally.
   d. If `waitingtofinish`: checks if all 8 channels are NULL; if so, sets `done = 1`.
4. Calls `quit()` after the loop exits.

---

## Data Structures

### `wadinfo_t` (local, mirrors `wadread.c`)

```c
typedef struct wadinfo_struct {
    char identification[4];  // Must be "IWAD"
    int  numlumps;            // Number of lumps in the WAD
    int  infotableofs;        // Byte offset to the lump directory
} wadinfo_t;
```

Declared locally inside `soundsrv.c` as a redundant copy of the same structure in `wadread.c`. The file comment calls this out: "Department of Redundancy Department." This copy is not actually used in `soundsrv.c` for any I/O; the actual WAD parsing is delegated to `wadread.c`.

### `filelump_t` (local, mirrors `wadread.c`)

```c
typedef struct filelump_struct {
    int  filepos;   // Byte offset of this lump in the WAD file
    int  size;      // Size of the lump in bytes
    char name[8];   // Up to 8-character lump name, not null-terminated
} filelump_t;
```

Also a redundant local copy. Not used in any `soundsrv.c` code paths; actual lump access goes through `wadread.c`.

---

## Dependencies

| Module | Usage |
|---|---|
| `sounds.h` | Declares `sfxenum_t` (all sfx IDs including `sfx_sawup`, etc.), `musicenum_t`, and `extern` arrays `S_sfx[]` / `S_music[]`. |
| `soundsrv.h` | Defines `SAMPLECOUNT`, `MIXBUFFERSIZE`, `SPEED`; declares `I_InitSound()`, `I_InitMusic()`, `I_SubmitOutputBuffer()`, `I_ShutdownSound()`, `I_ShutdownMusic()`. |
| `wadread.h` | Declares `openwad()` and `getsfx()` used in `grabdata()`. |
| `soundst.h` | (via `sounds.h`) Defines `sfxinfo_t` and `musicinfo_t` structs; `S_sfx[]` is an array of `sfxinfo_t`. |
| `linux.c` | Provides the platform implementations of `I_InitSound()`, `I_InitMusic()`, `I_SubmitOutputBuffer()`, `I_ShutdownSound()`, `I_ShutdownMusic()`. Also defines `sfxdevice` and `musdevice`. |
| Standard C / POSIX | `<math.h>` (pow), `<unistd.h>` (read, write, access, select), `<fcntl.h>` (open), `<sys/time.h>` (gettimeofday, struct timeval), `<stdlib.h>` (malloc, getenv, exit), `<stdio.h>` (fprintf, sprintf), `<sys/stat.h>` (fstat). |
