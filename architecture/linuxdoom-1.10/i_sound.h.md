# File Overview

`i_sound.h` is the system-level sound interface header for the DOOM engine. It defines the abstract platform-independent API that the higher-level sound manager (`s_sound.c`) calls to interact with audio hardware. The actual implementation of these functions lives in `i_sound.c`, which contains the platform-specific code for Linux/UNIX.

This header acts as the boundary between the portable game logic and the platform-specific audio backend. By keeping all sound hardware interactions behind this interface, DOOM can be ported to different platforms by simply replacing the implementation file while leaving all game code unchanged.

The file also contains a conditional section for the `SNDSERV` compile-time flag, which enables a separate sound server process mode (used on some UNIX systems where direct audio access is restricted).

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `FILE*` | `sndserver` | (SNDSERV only) File handle/pipe to the external sound server process |
| `char*` | `sndserver_filename` | (SNDSERV only) Filesystem path to the sound server executable |

Both variables are conditionally compiled under `#ifdef SNDSERV` and represent an alternative audio playback architecture where DOOM communicates with a separate dedicated sound process rather than accessing audio hardware directly.

## Functions

This file contains no function definitions — it is a header declaring the sound interface API. All functions listed here are implemented in `i_sound.c`.

### `void I_InitSound()`

**Signature:** `void I_InitSound()`

**Purpose:** Initializes the sound subsystem at program startup. Called once during engine initialization.

**Parameters:** None.

**Return value:** None.

**Key logic:** Implementation is platform-specific. On Linux, this initializes the audio hardware or establishes a connection to the sound server, sets up mixing channels, and prepares the engine to play sound effects.

---

### `void I_UpdateSound(void)`

**Signature:** `void I_UpdateSound(void)`

**Purpose:** Updates the sound buffer with newly mixed audio data. Called periodically during the game loop.

**Parameters:** None.

**Return value:** None.

**Key logic:** Mixes pending audio data into the output buffer. Part of the per-frame sound processing pipeline.

---

### `void I_SubmitSound(void)`

**Signature:** `void I_SubmitSound(void)`

**Purpose:** Submits the mixed sound buffer to the audio device for playback. Typically called after `I_UpdateSound`.

**Parameters:** None.

**Return value:** None.

**Key logic:** Triggers the actual hardware output of the previously mixed audio buffer.

---

### `void I_ShutdownSound(void)`

**Signature:** `void I_ShutdownSound(void)`

**Purpose:** Shuts down the sound effect subsystem at program termination.

**Parameters:** None.

**Return value:** None.

**Key logic:** Releases audio hardware resources, closes device handles, and frees any allocated sound buffers.

---

### `void I_SetChannels()`

**Signature:** `void I_SetChannels()`

**Purpose:** Initializes the sound channel structures. Called as part of the sound startup sequence.

**Parameters:** None.

**Return value:** None.

**Key logic:** Sets up the internal channel management data that tracks which sounds are playing on which output channels.

---

### `int I_GetSfxLumpNum(sfxinfo_t* sfxinfo)`

**Signature:** `int I_GetSfxLumpNum(sfxinfo_t* sfxinfo)`

**Purpose:** Returns the WAD lump index for a given sound effect descriptor.

**Parameters:**
- `sfxinfo` - Pointer to a `sfxinfo_t` structure describing the sound. Contains the sound name used to look up the WAD lump.

**Return value:** Integer lump number in the WAD file for the sound's raw data, or -1 if not found.

**Key logic:** Uses the sound name stored in `sfxinfo` to perform a WAD lump lookup via `W_GetNumForName`.

---

### `int I_StartSound(int id, int vol, int sep, int pitch, int priority)`

**Signature:** `int I_StartSound(int id, int vol, int sep, int pitch, int priority)`

**Purpose:** Starts playing a sound effect on a free channel and returns a handle to that playing instance.

**Parameters:**
- `id` - Sound effect ID (index into the sfx table, correlates to `sfxenum_t` values)
- `vol` - Volume level (0-127)
- `sep` - Stereo separation (0=hard left, 128=center, 255=hard right)
- `pitch` - Pitch shift value
- `priority` - Priority of this sound (higher priority sounds can preempt lower ones)

**Return value:** A handle (integer) identifying this particular playing sound instance, used by `I_StopSound`, `I_SoundIsPlaying`, and `I_UpdateSoundParams`. Returns -1 on failure.

**Key logic:** Finds a free (or lowest-priority occupied) mixing channel, loads the sound data into it, applies volume/separation/pitch, and begins playback.

---

### `void I_StopSound(int handle)`

**Signature:** `void I_StopSound(int handle)`

**Purpose:** Immediately stops the sound effect identified by the given handle.

**Parameters:**
- `handle` - Handle previously returned by `I_StartSound`

**Return value:** None.

**Key logic:** Locates the channel associated with the handle and halts playback.

---

### `int I_SoundIsPlaying(int handle)`

**Signature:** `int I_SoundIsPlaying(int handle)`

**Purpose:** Checks whether a sound channel is still actively playing.

**Parameters:**
- `handle` - Handle previously returned by `I_StartSound`

**Return value:** 1 if the sound is still playing, 0 if it has finished or the handle is invalid.

**Key logic:** Checks the state of the channel associated with the handle.

---

### `void I_UpdateSoundParams(int handle, int vol, int sep, int pitch)`

**Signature:** `void I_UpdateSoundParams(int handle, int vol, int sep, int pitch)`

**Purpose:** Updates the playback parameters (volume, stereo separation, pitch) of an already-playing sound. Called by `S_UpdateSounds` each tic to adjust 3D sound positioning.

**Parameters:**
- `handle` - Handle previously returned by `I_StartSound`
- `vol` - New volume (0-127)
- `sep` - New stereo separation (0-255)
- `pitch` - New pitch value

**Return value:** None.

**Key logic:** Locates the channel for the handle and updates its mixing parameters in real time.

---

### `void I_InitMusic(void)`

**Signature:** `void I_InitMusic(void)`

**Purpose:** Initializes the music playback subsystem.

**Parameters:** None.

**Return value:** None.

**Key logic:** Sets up the MIDI or OPL music system depending on the available hardware.

---

### `void I_ShutdownMusic(void)`

**Signature:** `void I_ShutdownMusic(void)`

**Purpose:** Shuts down the music subsystem at program exit.

**Parameters:** None.

**Return value:** None.

**Key logic:** Stops music playback and releases music hardware/software resources.

---

### `void I_SetMusicVolume(int volume)`

**Signature:** `void I_SetMusicVolume(int volume)`

**Purpose:** Sets the music playback volume.

**Parameters:**
- `volume` - Music volume level (0-15)

**Return value:** None.

**Key logic:** Passes the volume value to the underlying music hardware or software mixer.

---

### `void I_PauseSong(int handle)`

**Signature:** `void I_PauseSong(int handle)`

**Purpose:** Pauses a currently playing song (e.g., when the game is paused).

**Parameters:**
- `handle` - Music handle returned by `I_RegisterSong`

**Return value:** None.

---

### `void I_ResumeSong(int handle)`

**Signature:** `void I_ResumeSong(int handle)`

**Purpose:** Resumes a previously paused song.

**Parameters:**
- `handle` - Music handle returned by `I_RegisterSong`

**Return value:** None.

---

### `int I_RegisterSong(void *data)`

**Signature:** `int I_RegisterSong(void *data)`

**Purpose:** Registers raw music data with the sound system and prepares it for playback.

**Parameters:**
- `data` - Pointer to raw MUS or MIDI music data loaded from the WAD

**Return value:** An integer handle identifying this registered song, used by subsequent `I_PlaySong`, `I_StopSong`, and `I_UnRegisterSong` calls.

**Key logic:** Parses the music data and prepares it for the MIDI/OPL playback engine.

---

### `void I_PlaySong(int handle, int looping)`

**Signature:** `void I_PlaySong(int handle, int looping)`

**Purpose:** Begins playback of a previously registered song.

**Parameters:**
- `handle` - Music handle returned by `I_RegisterSong`
- `looping` - If non-zero, the song loops continuously; if zero, it plays once

**Return value:** None.

**Key logic:** Starts the music playback engine with the specified song, configuring it to loop or play once based on the `looping` parameter.

---

### `void I_StopSong(int handle)`

**Signature:** `void I_StopSong(int handle)`

**Purpose:** Stops a playing song, fading out over approximately 3 seconds (per the comment, though implementation may vary).

**Parameters:**
- `handle` - Music handle returned by `I_RegisterSong`

**Return value:** None.

---

### `void I_UnRegisterSong(int handle)`

**Signature:** `void I_UnRegisterSong(int handle)`

**Purpose:** Unregisters a song and frees any resources associated with it.

**Parameters:**
- `handle` - Music handle returned by `I_RegisterSong`

**Return value:** None.

**Key logic:** Inverse of `I_RegisterSong`. Releases memory and handles associated with the song data.

## Data Structures

No new data structures are defined in this file. It uses `sfxinfo_t` from `sounds.h`.

## Dependencies

| File | Reason |
|------|--------|
| `doomdef.h` | Basic DOOM type definitions and constants |
| `doomstat.h` | Game state variables (needed for some sound decisions) |
| `sounds.h` | `sfxinfo_t` type definition and sound enumeration (`sfxenum_t`) |
| `<stdio.h>` | Only included under `#ifdef SNDSERV` for the `FILE*` type used by `sndserver` |
