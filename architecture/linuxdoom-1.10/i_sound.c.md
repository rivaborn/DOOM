# File Overview

`i_sound.c` is the Linux/Unix platform-specific sound implementation for DOOM. It provides two alternative sound output modes controlled by compile-time flags: `SNDSERV` mode (delegates audio to a separate `sndserver` process via a pipe) and native OSS (Open Sound System) mode (direct `/dev/dsp` access). In OSS mode, this file implements a full software mixer: it loads all sound effects from the WAD at startup, manages 8 simultaneous mixing channels, mixes 16-bit stereo PCM audio from multiple active sounds, applies per-channel volume and stereo separation, and writes the mixed buffer to the audio device. The music API is present but entirely unimplemented — all music functions are stubs. An optional `SNDINTR` build mode (timer-driven interrupt output) is also scaffolded but not the default.

## Global Variables

### Sound Buffer and Device

| Type | Name | Description |
|------|------|-------------|
| `int` | `audio_fd` | File descriptor for the `/dev/dsp` OSS audio device. |
| `signed short` | `mixbuffer[MIXBUFFERSIZE]` | The global 16-bit stereo mixing buffer. Interleaved left/right samples. Size is `SAMPLECOUNT * BUFMUL` = 512 * 4 = 2048 shorts. |
| `int` | `lengths[NUMSFX]` | Actual padded byte length of each loaded sound effect's raw PCM data. |

### Mixer Channel State (8 channels, indexed 0–7)

| Type | Name | Description |
|------|------|-------------|
| `unsigned int` | `channelstep[NUM_CHANNELS]` | Playback step rate for each channel (for pitch adjustment). Currently set but not meaningfully used by the mixer. |
| `unsigned int` | `channelstepremainder[NUM_CHANNELS]` | Sub-sample remainder for step accumulation. |
| `unsigned char*` | `channels[NUM_CHANNELS]` | Pointer to the current read position in each channel's PCM data. NULL when the channel is idle. |
| `unsigned char*` | `channelsend[NUM_CHANNELS]` | Pointer to the end of each channel's PCM data. |
| `int` | `channelstart[NUM_CHANNELS]` | `gametic` value when each channel started playing. Used to evict the oldest channel when all 8 are busy. |
| `int` | `channelhandles[NUM_CHANNELS]` | Handle number assigned when the sound started. |
| `int` | `channelids[NUM_CHANNELS]` | SFX enum ID of the sound playing in each channel. Used to prevent duplicate chainsaw/pistol sounds. |

### Volume and Pitch Tables

| Type | Name | Description |
|------|------|-------------|
| `int` | `steptable[256]` | Pitch-to-step lookup table (power-of-2 scaling). Computed in `I_SetChannels` but described as "currently unused." |
| `int` | `vol_lookup[128*256]` | Volume lookup table. `vol_lookup[vol * 256 + sample]` gives the scaled, sign-converted output sample for a given volume (0–127) and raw unsigned sample byte (0–255). Converts unsigned 8-bit samples to signed output by subtracting 128 and scaling. |
| `int*` | `channelleftvol_lookup[NUM_CHANNELS]` | Per-channel pointer into `vol_lookup` for the left channel volume. |
| `int*` | `channelrightvol_lookup[NUM_CHANNELS]` | Per-channel pointer into `vol_lookup` for the right channel volume. |

### SNDSERV Mode

| Type | Name | Description |
|------|------|-------------|
| `FILE*` | `sndserver` | Pipe to the external sound server process (SNDSERV mode only). |
| `char*` | `sndserver_filename` | Path to the sound server executable, default `"./sndserver "`. |

### Timer Interrupt State (SNDINTR mode)

| Type | Name | Description |
|------|------|-------------|
| `static int` | `flag` | Synchronization flag between the mixing step and the asynchronous audio write interrupt. |
| `static int` | `itimer` | Timer type (`ITIMER_REAL` by default). |
| `static int` | `sig` | Signal number (`SIGALRM`) for the audio timer interrupt. |
| `static int` | `looping` | Music loop flag (stub, always 0). |
| `static int` | `musicdies` | Gametic at which the current "music" ends (stub). |

### Build Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `SAMPLECOUNT` | `512` | Number of samples per mix buffer. |
| `NUM_CHANNELS` | `8` | Number of simultaneous sound channels. |
| `BUFMUL` | `4` | Buffer size multiplier (2 for 16-bit × 2 for stereo). |
| `MIXBUFFERSIZE` | `SAMPLECOUNT * BUFMUL` = 2048 | Total shorts in `mixbuffer`. |
| `SAMPLERATE` | `11025` | Audio sample rate in Hz. |
| `SAMPLESIZE` | `2` | Bytes per sample (16-bit). |

## Functions

### `myioctl`

```c
void myioctl(int fd, int command, int* arg)
```

**Purpose:** Convenience wrapper around `ioctl` that calls `I_Error` on failure.

**Parameters:**
- `fd` — File descriptor.
- `command` — `ioctl` command code.
- `arg` — Pointer to the argument.

**Returns:** Nothing.

---

### `getsfx`

```c
void* getsfx(char* sfxname, int* len)
```

**Purpose:** Loads a single sound effect from the WAD, pads it to a multiple of `SAMPLECOUNT` bytes (so the mixer never overruns), and returns a pointer to the raw PCM data (offset past the 8-byte DOOM sound header).

**Parameters:**
- `sfxname` — The base name of the sound (without the `"ds"` prefix). E.g., `"pistol"` → loads lump `"dspistol"`.
- `len` — Output parameter set to the padded data length.

**Returns:** Pointer to the padded PCM data (8 bytes past the WAD lump start).

**Key logic:**
- Constructs the lump name by prepending `"ds"` to `sfxname`.
- Falls back to `"dspistol"` if the lump doesn't exist (handles DOOM II sounds in shareware mode).
- Pads raw data to the next multiple of `SAMPLECOUNT` with value `128` (silence for unsigned 8-bit audio).
- Allocates from zone memory with `PU_STATIC` tag; frees the original cached lump.

---

### `addsfx`

```c
int addsfx(int sfxid, int volume, int step, int seperation)
```

**Purpose:** Adds a sound effect to the mixer by claiming one of the 8 channels. Handles duplicate-sound suppression for special effects (chainsaw, pistol). Calculates per-channel left/right volume lookup table pointers based on stereo separation.

**Parameters:**
- `sfxid` — SFX enum index.
- `volume` — Volume level (0–127).
- `step` — Playback step rate.
- `seperation` — Stereo separation (0–255, where 128 is center).

**Returns:** A handle number for the started sound (currently not used to stop sounds).

**Key logic:**
- Suppresses duplicate instances of chainsaw and pistol sounds by stopping any existing channel playing those SFX IDs.
- Finds the oldest active channel (by `channelstart[]`) to evict if all 8 are busy.
- Sets channel data pointers, handle, step, start time.
- Computes left/right volumes using an `x^2` stereo pan formula:
  ```
  leftvol  = volume - (volume * sep * sep) >> 16
  rightvol = volume - (volume * (sep-257) * (sep-257)) >> 16
  ```
- Sets `channelleftvol_lookup[slot]` and `channelrightvol_lookup[slot]` to point into `vol_lookup`.

---

### `I_SetChannels`

```c
void I_SetChannels(void)
```

**Purpose:** Initializes the pitch step table (`steptable`) and the volume lookup table (`vol_lookup`). Called by the sound system initialization.

**Parameters:** None.
**Returns:** Nothing.

**Key logic:**
- `steptable[128+i] = (int)(pow(2.0, i/64.0) * 65536.0)` — Power-of-2 pitch scaling table (acknowledged as unused).
- `vol_lookup[i*256+j] = (i * (j-128) * 256) / 127` — Volume × signed-sample lookup. Converts unsigned 8-bit sample `j` (0–255) to signed by subtracting 128, then scales by volume `i` (0–127).

---

### `I_SetSfxVolume`

```c
void I_SetSfxVolume(int volume)
```

**Purpose:** Sets the global sound effects volume by updating `snd_SfxVolume`.

**Parameters:**
- `volume` — New SFX volume (0–15 typically).

---

### `I_SetMusicVolume`

```c
void I_SetMusicVolume(int volume)
```

**Purpose:** Sets the global music volume (stub — no actual music output is implemented). Updates `snd_MusicVolume`.

---

### `I_GetSfxLumpNum`

```c
int I_GetSfxLumpNum(sfxinfo_t* sfx)
```

**Purpose:** Returns the WAD lump number for a given sound effect by constructing the `"ds"` + name lump name.

**Parameters:**
- `sfx` — Pointer to the `sfxinfo_t` for the sound.

**Returns:** WAD lump index.

---

### `I_StartSound`

```c
int I_StartSound(int id, int vol, int sep, int pitch, int priority)
```

**Purpose:** Starts playing a sound effect. In SNDSERV mode, writes a command to the sound server pipe. In OSS mode, calls `addsfx` to claim a mixer channel.

**Parameters:**
- `id` — SFX enum ID.
- `vol` — Volume (0–127).
- `sep` — Stereo separation (0–255).
- `pitch` — Pitch index (indexes into `steptable`).
- `priority` — Priority (currently ignored/unused).

**Returns:** Sound handle (the `id` in SNDSERV mode; the channel handle in OSS mode).

---

### `I_StopSound`

```c
void I_StopSound(int handle)
```

**Purpose:** Stops a playing sound. Currently unimplemented — the handle parameter is set to 0 and discarded. Stopping sounds by handle is described as requiring a channel scan.

---

### `I_SoundIsPlaying`

```c
int I_SoundIsPlaying(int handle)
```

**Purpose:** Returns whether a sound (identified by handle) is still playing. The implementation uses the handle as a `gametic` expiry value (`return gametic < handle`) — a noted hack ("Ouch").

---

### `I_UpdateSound`

```c
void I_UpdateSound(void)
```

**Purpose:** The core software mixing function. Iterates over all `SAMPLECOUNT` (512) sample pairs, summing contributions from all active channels, applying per-channel volume lookup, advancing channel read pointers, and clamping the result to the 16-bit signed range before storing in `mixbuffer`.

**Parameters:** None.
**Returns:** Nothing.

**Key logic:**
- Interleaved left/right output: `mixbuffer[0]` = left sample 0, `mixbuffer[1]` = right sample 0, etc.
- For each active channel (`channels[chan] != NULL`):
  - Reads one unsigned byte sample.
  - Adds `channelleftvol_lookup[chan][sample]` to the left accumulator `dl`.
  - Adds `channelrightvol_lookup[chan][sample]` to the right accumulator `dr`.
  - Advances the channel's read pointer by `channelstepremainder[chan] >> 16` bytes.
  - Clears the channel pointer when the end is reached.
- Clamps `dl` and `dr` to `[-0x8000, 0x7fff]` and stores in `mixbuffer`.

---

### `I_SubmitSound`

```c
void I_SubmitSound(void)
```

**Purpose:** Writes the completed `mixbuffer` to the OSS audio device (`/dev/dsp`).

**Parameters:** None.
**Returns:** Nothing.

---

### `I_UpdateSoundParams`

```c
void I_UpdateSoundParams(int handle, int vol, int sep, int pitch)
```

**Purpose:** Updates the volume/separation/pitch of a playing sound. Entirely unimplemented — all parameters are set to 0 and discarded.

---

### `I_ShutdownSound`

```c
void I_ShutdownSound(void)
```

**Purpose:** Shuts down the sound system. In SNDSERV mode, sends a quit command. In OSS mode, waits (incompletely) for pending sounds and closes `/dev/dsp`.

---

### `I_InitSound`

```c
void I_InitSound(void)
```

**Purpose:** Initializes the entire sound system. In SNDSERV mode, launches the sound server process. In OSS mode: opens `/dev/dsp`, configures it for 11025 Hz stereo 16-bit output, pre-loads all sound effects from the WAD, and zeroes the mix buffer.

**Key logic (OSS mode):**
1. Opens `/dev/dsp` with `O_WRONLY`.
2. Configures fragment size, resets DSP, sets sample rate to 11025 Hz, enables stereo, verifies `AFMT_S16_LE` format support.
3. Iterates `S_sfx[1..NUMSFX-1]`: loads non-aliased sounds via `getsfx`; for aliased sounds (e.g., chaingun linked to pistol), copies the pointer and length from the linked entry.
4. Zeroes `mixbuffer`.

### Music Stubs

All music functions are empty stubs:

| Function | Behavior |
|----------|----------|
| `I_InitMusic` | No-op. |
| `I_ShutdownMusic` | No-op. |
| `I_PlaySong(handle, looping)` | Sets `musicdies = gametic + TICRATE*30`. |
| `I_PauseSong(handle)` | No-op. |
| `I_ResumeSong(handle)` | No-op. |
| `I_StopSong(handle)` | Clears `looping`, sets `musicdies = 0`. |
| `I_UnRegisterSong(handle)` | No-op. |
| `I_RegisterSong(data)` | Returns `1`. |
| `I_QrySongPlaying(handle)` | Returns `looping || musicdies > gametic`. |

### Timer Interrupt Functions (SNDINTR mode only)

| Function | Description |
|----------|-------------|
| `I_HandleSoundTimer(int ignore)` | SIGALRM signal handler; writes `mixbuffer` to audio device if `flag` is set. |
| `I_SoundSetTimer(int duration_of_tick)` | Sets up `setitimer` with SIGALRM to call `I_HandleSoundTimer` at regular intervals. |
| `I_SoundDelTimer(void)` | Removes the timer by calling `I_SoundSetTimer(0)`. |

## Dependencies

- `stdio.h`, `stdlib.h`, `stdarg.h`, `math.h` — Standard C library (math for `pow`).
- `sys/time.h`, `sys/types.h`, `fcntl.h`, `unistd.h`, `sys/ioctl.h` — POSIX system calls.
- `linux/soundcard.h` — OSS (Open Sound System) ioctls for `/dev/dsp` configuration.
- `time.h`, `signal.h` — Timer and signal APIs for SNDINTR mode.
- `z_zone.h` — `Z_Malloc`, `Z_Free` for zone memory allocation of sound data.
- `i_system.h` — `I_Error` for fatal errors.
- `i_sound.h` — Own header.
- `m_argv.h` — Command-line argument access (currently unused in this file).
- `m_misc.h` — Miscellaneous utilities.
- `w_wad.h` — `W_CacheLumpName`, `W_CacheLumpNum`, `W_GetNumForName`, `W_CheckNumForName`, `W_LumpLength`.
- `doomdef.h` — Core type definitions.
- `doomstat.h` — `gametic`, `snd_SfxVolume`, `snd_MusicVolume`.
- `sounds.h` — `S_sfx[]` sound info array, `sfxinfo_t`, `NUMSFX`.
