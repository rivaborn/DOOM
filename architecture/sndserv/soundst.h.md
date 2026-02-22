# File Overview

**Path:** `sndserv/soundst.h`
**RCS:** `$Id: soundst.h,v 1.3 1997/01/29 22:40:45 b1 Exp $`

`soundst.h` ("sound structures") is the foundational type-definition header for DOOM's audio system. It defines the core data structures representing sound effects and music tracks, declares all the high-level `S_*` (sound module) and low-level `I_*` (implementation/platform) function interfaces, and provides the audio-related constants used across both the main engine and the sound server process.

This header is included by `sounds.h`, which is in turn included by `soundsrv.c`. It is also shared (conceptually identical) with the main engine in `linuxdoom-1.10/`.

The comment *"See gensounds.h and gensounds.c for what soundst.h is made of"* refers to a code-generation tool (not included in the release) that originally produced this file from a higher-level description.

---

## Global Variables

Declared as `extern` (defined in `sndserv/sounds.c`):

| Type | Name | Purpose |
|---|---|---|
| `sfxinfo_t[]` | `S_sfx` | Master array of all sound effect descriptors, indexed by `sfxenum_t`. |
| `musicinfo_t[]` | `S_music` | Master array of all music track descriptors, indexed by `musicenum_t`. |

---

## Functions (Declarations)

This header declares two layers of audio API:

### High-Level Sound Module (`S_*` functions)

These are the game-logic-facing interface. In the main DOOM engine they are implemented in `s_sound.c`. They are declared here for reference but **not implemented or called within the sound server process itself** — the sound server uses the lower-level `I_*` interface directly.

| Signature | Purpose |
|---|---|
| `void S_Start(void)` | Initialize the sound code at the start of a level. |
| `void S_StartSound(void* origin, int sound_id)` | Start a sound effect at the position of an in-game object. |
| `void S_StartSoundAtVolume(void* origin, int sound_id, int volume)` | Start a sound effect at a specified volume. |
| `void S_StopSound(void* origin)` | Stop any sound currently associated with a given origin object. |
| `void S_StartMusic(int music_id)` | Start a music track by ID. |
| `void S_ChangeMusic(int music_id, int looping)` | Switch to a new music track; specify whether to loop. |
| `void S_StopMusic(void)` | Stop currently playing music. |
| `void S_PauseSound(void)` | Pause all sound output. |
| `void S_ResumeSound(void)` | Resume paused sound output. |
| `void S_UpdateSounds(void* listener)` | Per-frame update: recomputes volumes and pans based on listener position. |
| `void S_SetMusicVolume(int volume)` | Set the music volume level. |
| `void S_SetSfxVolume(int volume)` | Set the sound effects volume level. |
| `void S_Init(int, int)` | Initialize the sound system including volume. |

### Low-Level Platform Interface (`I_*` functions)

These are implemented by the platform layer (`linux.c` in the sound server). They are called from within `soundsrv.c` and `linux.c`.

| Signature | Purpose |
|---|---|
| `void I_SetMusicVolume(int volume)` | Set music volume at the hardware/driver level. |
| `void I_SetSfxVolume(int volume)` | Set SFX volume at the hardware/driver level. |
| `void I_PauseSong(int handle)` | Pause a registered song. |
| `void I_ResumeSong(int handle)` | Resume a paused song. |
| `void I_PlaySong(int handle, int looping)` | Begin playing a registered song; loop if `looping` is non-zero. |
| `void I_StopSong(int handle)` | Stop a song (fades out over 3 seconds per the comment). |
| `int I_RegisterSong(void* data)` | Register a block of music data (MUS format) with the music driver; returns a handle. |
| `void I_UnRegisterSong(int handle)` | Release a previously registered song handle. |
| `int I_QrySongPlaying(int handle)` | Query whether a song is currently playing. |
| `void I_SetChannels(int channels)` | Set the number of mixing channels. |
| `int I_GetSfxLumpNum(sfxinfo_t*)` | Get the WAD lump number for a sound effect. |
| `int I_StartSound(int id, void* data, int vol, int sep, int pitch, int priority)` | Start a sound effect; returns a channel handle. |
| `void I_UpdateSoundParams(int handle, int vol, int sep, int pitch)` | Update volume, stereo separation, and pitch of an active channel. |
| `void I_StopSound(int handle)` | Stop a specific sound channel. |
| `int I_SoundIsPlaying(int handle)` | Returns `1` if the channel is still active, `0` otherwise. |

---

## Data Structures

### `musicinfo_t`

```c
typedef struct {
    char* name;    // Up to 6-character music track name (WAD lump name without "D_" prefix)
    int   lumpnum; // WAD lump number; -1 until looked up
    void* data;    // Pointer to loaded MUS format data; NULL until loaded
    int   handle;  // Driver handle once registered with I_RegisterSong()
} musicinfo_t;
```

Describes a single music track. The array `S_music[]` in `sounds.c` is statically initialized with names only; `lumpnum`, `data`, and `handle` are filled in at runtime by the sound module.

---

### `sfxinfo_t` (self-referential struct)

```c
typedef struct sfxinfo_struct sfxinfo_t;

struct sfxinfo_struct {
    char*      name;        // Up to 6-character WAD name suffix (e.g., "pistol" -> lump "dspistol")
    int        singularity; // Non-zero if only one instance can play at a time
    int        priority;    // Numeric priority; higher = less likely to be evicted
    sfxinfo_t* link;        // If non-NULL, this is an alias for another sfx entry
    int        pitch;       // Pitch modifier when played as a link/alias
    int        volume;      // Volume modifier when played as a link/alias
    void*      data;        // Pointer to loaded PCM sample data (set by grabdata() in soundsrv.c)
    int        usefulness;  // Reference count: >0 in use, 0 decrement, -1 evict
    int        lumpnum;     // WAD lump number; cached after first lookup
};
```

The central descriptor for a sound effect. Key design notes:

- **`link`**: Allows multiple `sfxenum_t` IDs to share the same PCM data. In `grabdata()`, if `S_sfx[i].link != NULL`, the `data` pointer and `lengths[]` entry are copied from the linked entry instead of loading a new lump. This saves both WAD reads and memory for sounds that are pitch variations of a base sample.
- **`singularity`**: Checked in `addsfx()` to stop any currently playing instance of the same sound before starting a new one. However, in practice `soundsrv.c` hard-codes its own list of singular sounds (`sfx_sawup`, `sfx_sawidl`, `sfx_sawful`, `sfx_sawhit`, `sfx_stnmov`, `sfx_pistol`) rather than reading this field.
- **`usefulness`**: Intended for a cache-eviction scheme where sounds not recently played can be unloaded. Not actively used in the sound server (all sounds are preloaded and kept).

---

### `channel_t`

```c
typedef struct {
    sfxinfo_t* sfxinfo;  // Sound info for the playing sound; NULL = channel free
    void*      origin;   // In-game object (mobj_t*) that is the source of this sound
    int        handle;   // Channel handle as returned by I_StartSound()
} channel_t;
```

High-level channel descriptor used by the main engine's `s_sound.c` to track what is playing in each logical channel. **Not used within the sound server process** — the sound server manages channels directly via the parallel arrays in `soundsrv.c` (`channels[]`, `channelids[]`, etc.).

---

## Preprocessor Constants

| Constant | Value | Purpose |
|---|---|---|
| `S_MAX_VOLUME` | `127` | Maximum volume level for sound effects (0–127 range). |
| `S_CLIPPING_DIST` | `1200 * 0x10000` | Fixed-point distance beyond which sounds are completely inaudible (clipped out). In DOOM map units (1 unit ≈ pixel scale). |
| `S_CLOSE_DIST` | `200 * 0x10000` | Fixed-point distance within which sounds play at maximum volume. |
| `S_ATTENUATOR` | `(S_CLIPPING_DIST - S_CLOSE_DIST) >> FRACBITS` | Attenuation divisor; the integer distance range over which volume scales from max to zero. |
| `NORM_PITCH` | `128` | Default pitch (index 128 in `steptable[]` = `1.0x` = native sample rate). |
| `NORM_PRIORITY` | `64` | Default sound priority level. |
| `NORM_VOLUME` | `snd_MaxVolume` | Default volume (evaluates to the current max volume setting). |
| `S_PITCH_PERTURB` | `1` | Whether pitch perturbation (randomization) is enabled. |
| `NORM_SEP` | `128` | Default stereo separation value (center). |
| `S_STEREO_SWING` | `96 * 0x10000` | Maximum stereo separation swing in fixed-point map units. |
| `S_IFRACVOL` | `30` | Percentage attenuation from front to back. |
| `NA` | `0` | "Not applicable" placeholder, used in `S_sfx[]` initializers for unused fields. |
| `S_NUMCHANNELS` | `2` | Default number of audio channels in the high-level sound module (not the same as the 8 hardware mixing channels). |
| `FREQ_LOW` | `0x40` | Low frequency flag constant. |
| `FREQ_NORM` | `0x80` | Normal frequency flag constant. |
| `FREQ_HIGH` | `0xFF` | High frequency flag constant. |

---

## Anonymous Enumerations

### Sound type discriminators
```c
enum { Music, Sfx, SfxLink };
```
Used to classify entries in the master sound table (Music = music track, Sfx = normal sound, SfxLink = alias pointing to another sound).

### Audio card type flags (bitmask)
```c
enum { PC=1, Adlib=2, SB=4, Midi=8 };
```
Bitmask values identifying which audio hardware is available or selected. `PC` = PC speaker, `Adlib` = OPL FM synthesis, `SB` = Sound Blaster PCM, `Midi` = MIDI interface.

### SFX cache state values
```c
enum { sfxThrowOut=-1, sfxNotUsed=0 };
```
Sentinel values for `sfxinfo_t.usefulness`: `-1` means the sound data can be freed; `0` means it is currently unused but retained.

---

## Include Guard

```c
#ifndef __SOUNDSTH__
#define __SOUNDSTH__
// ...
#endif
```

---

## Dependencies

| Module | Usage |
|---|---|
| None | This header includes no other headers. It defines foundational types and therefore sits at the bottom of the include hierarchy. |

Files that include `soundst.h`:

| File | Why |
|---|---|
| `sounds.h` | Includes `soundst.h` to get `sfxinfo_t` and `musicinfo_t` before declaring `extern sfxinfo_t S_sfx[]` and `extern musicinfo_t S_music[]`. |
| (Implicitly) `soundsrv.c` | Via `sounds.h` — needs `sfxinfo_t` to access `S_sfx[i].data`, `.name`, `.link`. |
