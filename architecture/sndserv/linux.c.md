# File Overview

`linux.c` is the Linux platform-specific audio backend for the DOOM sound server (`sndserv`). It implements the hardware abstraction layer (HAL) for audio output on Linux systems, using the OSS (Open Sound System) kernel interface exposed via `/dev/dsp`.

The sound server (`soundsrv.c`) is designed to be platform-portable: it performs all sound mixing in software and then delegates actual audio output to a small set of platform functions. `linux.c` provides the Linux implementations of those functions. It is compiled as part of the `sndserv` executable, which runs as a separate process from the main DOOM game — DOOM communicates with it via a pipe on stdin/stdout.

The file configures the audio device for:
- 11025 Hz sample rate (the standard DOOM mixing rate)
- Stereo output (2 channels)
- 16-bit signed little-endian samples (`AFMT_S16_LE`)
- Fragment size of 2^11 = 2048 bytes with 2 fragments

Music support is stubbed out (empty function bodies) since this server only handles sound effects.

RCS version: `$Id: linux.c,v 1.3 1997/01/26 07:45:01 b1 Exp $`

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `int` | `audio_fd` | File descriptor for the `/dev/dsp` OSS audio device. Opened in `I_InitSound`, closed in `I_ShutdownSound`. Used by `I_SubmitOutputBuffer` to write mixed audio data. |

---

## Functions

### `myioctl`

**Signature:**
```c
void myioctl(int fd, int command, int *arg)
```

**Purpose:**
A wrapper around the standard `ioctl` system call that adds error checking and terminates the program on failure. Used exclusively during audio device initialization in `I_InitSound`.

**Parameters:**
- `fd` - File descriptor of the device to control (the `/dev/dsp` fd).
- `command` - The ioctl command code (e.g., `SNDCTL_DSP_SPEED`, `SNDCTL_DSP_STEREO`).
- `arg` - Pointer to an integer argument; serves as both input and output depending on the command.

**Return Value:**
None (void). On failure, prints diagnostic messages to `stderr` (including the `errno` value) and calls `exit(-1)`.

**Key Logic:**
Calls `ioctl(fd, command, arg)`. If the return value `rc` is negative (indicating an error), prints the failed command number and the system `errno`, then exits. The `errno` variable is explicitly declared `extern` in function scope, which is the pre-ANSI C way of accessing it.

---

### `I_InitMusic`

**Signature:**
```c
void I_InitMusic(void)
```

**Purpose:**
Stub function for music initialization. No music playback is implemented in the Linux sound server.

**Parameters:**
None.

**Return Value:**
None (void). Empty function body.

---

### `I_InitSound`

**Signature:**
```c
void I_InitSound(int samplerate, int samplesize)
```

**Purpose:**
Initializes the Linux OSS audio device for sound effect playback. Opens `/dev/dsp` and configures it for the format expected by the DOOM sound mixer.

**Parameters:**
- `samplerate` - The desired sample rate in Hz. The function ignores this parameter and hardcodes 11025 Hz.
- `samplesize` - The desired sample size in bits. The function ignores this parameter and hardcodes 16-bit signed samples.

**Return Value:**
None (void).

**Key Logic:**

1. **Open device:** Opens `/dev/dsp` with `O_WRONLY`. Prints an error to `stderr` if it fails (but does not exit — the caller may still proceed).

2. **Set fragment size:** Calls `myioctl` with `SNDCTL_DSP_SETFRAGMENT` and the value `11 | (2<<16)`. This encodes: low 16 bits = 11 (fragment size = 2^11 = 2048 bytes), high 16 bits = 2 (number of fragments = 2). Total buffer = 4096 bytes.

3. **Reset DSP:** Calls `SNDCTL_DSP_RESET` to clear any previous state.

4. **Set sample rate:** Sets `i = 11025` and calls `SNDCTL_DSP_SPEED`. The actual rate set by the driver may differ; the return value is ignored.

5. **Set stereo:** Sets `i = 1` (stereo) and calls `SNDCTL_DSP_STEREO`.

6. **Set format:** Queries supported formats with `SNDCTL_DSP_GETFMTS`, then checks if `AFMT_S16_LE` (16-bit signed little-endian) is among them. If so, selects it with `SNDCTL_DSP_SETFMT`. Otherwise prints an error to `stderr` but does not abort.

---

### `I_SubmitOutputBuffer`

**Signature:**
```c
void I_SubmitOutputBuffer(void *samples, int samplecount)
```

**Purpose:**
Writes a buffer of mixed stereo audio samples to the OSS audio device for playback.

**Parameters:**
- `samples` - Pointer to the mixed audio data (stereo 16-bit samples interleaved as left, right, left, right...).
- `samplecount` - Number of stereo sample pairs to write.

**Return Value:**
None (void).

**Key Logic:**
Calls `write(audio_fd, samples, samplecount * 4)`. The factor of 4 accounts for stereo (2 channels) times 2 bytes per 16-bit sample. So `samplecount` is the number of stereo frames, and `samplecount * 4` is the total byte count.

---

### `I_ShutdownSound`

**Signature:**
```c
void I_ShutdownSound(void)
```

**Purpose:**
Closes the audio device file descriptor, releasing the `/dev/dsp` resource.

**Parameters:**
None.

**Return Value:**
None (void).

**Key Logic:**
Calls `close(audio_fd)`.

---

### `I_ShutdownMusic`

**Signature:**
```c
void I_ShutdownMusic(void)
```

**Purpose:**
Stub function for music shutdown. No music playback is implemented.

**Parameters:**
None.

**Return Value:**
None (void). Empty function body.

---

## Data Structures

None defined in this file.

---

## Dependencies

| File | Reason |
|------|--------|
| `<stdlib.h>` | `exit` |
| `<stdio.h>` | `fprintf` |
| `<fcntl.h>` | `open`, `O_WRONLY` |
| `<unistd.h>` | `write`, `close`, `ioctl` (transitively) |
| `<linux/soundcard.h>` | OSS audio ioctl constants (`SNDCTL_DSP_*`, `AFMT_S16_LE`) |
| `soundsrv.h` | `SAMPLECOUNT`, `MIXBUFFERSIZE`, `SPEED` constants; `I_InitSound`, `I_SubmitOutputBuffer`, `I_ShutdownSound`, `I_InitMusic`, `I_ShutdownMusic` function prototypes |
