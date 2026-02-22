# File Overview

**Path:** `sndserv/soundsrv.h`
**RCS:** `$Id: soundsrv.h,v 1.3 1997/01/29 22:40:44 b1 Exp $`

`soundsrv.h` is the central configuration and platform-abstraction header for the DOOM sound server (`sndserv`). It defines the audio buffer constants that drive both the mixer and the hardware output, and it declares the platform-specific I/O interface that `soundsrv.c` calls to interact with the actual audio device.

This header is included by every source file in the `sndserv` directory (`soundsrv.c`, `wadread.c`, and `linux.c`). It forms the glue layer between the platform-independent mixer logic in `soundsrv.c` and the Linux OSS implementation in `linux.c`.

---

## Global Variables

None. This header declares no global variables.

---

## Functions (Declarations)

All functions declared here are implemented in `linux.c` (or whatever platform-specific source provides the audio backend).

### `void I_InitMusic(void)`

**Purpose:** Initialize the music subsystem. In the Linux implementation, this is typically a stub or performs minimal setup since MIDI music is not handled by the sound server PCM pipeline.

**Parameters:** None.

**Returns:** void.

---

### `void I_InitSound(int samplerate, int samplesound)`

**Purpose:** Open and configure the audio output device for PCM sound effect playback.

**Parameters:**
- `samplerate` - Desired sample rate in Hz. Called from `main()` with `11025`.
- `samplesound` - Bit depth of each sample. Called from `main()` with `16`.

**Returns:** void.

---

### `void I_SubmitOutputBuffer(void* samples, int samplecount)`

**Purpose:** Write a completed buffer of mixed PCM audio to the hardware device. Called once per main-loop iteration from `updatesounds()` after `mix()` has filled `mixbuffer`.

**Parameters:**
- `samples` - Pointer to the mixed stereo 16-bit PCM buffer (i.e., `mixbuffer`).
- `samplecount` - Number of stereo sample *frames* (not bytes) to write. Called with `SAMPLECOUNT` (512).

**Returns:** void.

---

### `void I_ShutdownSound(void)`

**Purpose:** Close and release the PCM audio device. Called from `quit()` during clean shutdown.

**Parameters:** None.

**Returns:** void.

---

### `void I_ShutdownMusic(void)`

**Purpose:** Shut down the music subsystem. Called from `quit()` before `I_ShutdownSound()`.

**Parameters:** None.

**Returns:** void.

---

## Data Structures

None. This header defines no structs or typedefs.

---

## Preprocessor Constants

| Constant | Value | Purpose |
|---|---|---|
| `SAMPLECOUNT` | `512` | Number of stereo sample frames mixed and submitted per main-loop tick. At 11025 Hz this corresponds to approximately 46.4 ms of audio per tick. |
| `MIXBUFFERSIZE` | `SAMPLECOUNT * 2 * 2` = `2048` | Total size of `mixbuffer` in bytes: 512 samples × 2 channels (stereo) × 2 bytes per sample (16-bit). |
| `SPEED` | `11025` | Nominal sample rate in Hz. Matches the value passed to `I_InitSound()`. Declared but not used directly in any expression within the sound server source (the literal `11025` is used at the call site in `main()`). |

---

## Include Guard

```c
#ifndef __SNDSERVER_H__
#define __SNDSERVER_H__
// ...
#endif
```

---

## Dependencies

| Module | Usage |
|---|---|
| None | This header has no `#include` directives. It is a pure interface declaration. |

Files that include this header:

| File | Why |
|---|---|
| `soundsrv.c` | Uses `SAMPLECOUNT`, `MIXBUFFERSIZE`; calls `I_InitSound()`, `I_InitMusic()`, `I_SubmitOutputBuffer()`, `I_ShutdownSound()`, `I_ShutdownMusic()`. |
| `wadread.c` | Includes it for the `SAMPLECOUNT` constant used in `getsfx()` to compute the padded sound-data size. |
| `linux.c` | Includes it to implement the declared function prototypes and to reference `SAMPLECOUNT`/`MIXBUFFERSIZE` for device configuration and buffer sizing. |
