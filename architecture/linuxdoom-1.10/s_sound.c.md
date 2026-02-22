# File Overview

`s_sound.c` is the platform-independent sound manager for the DOOM engine. It sits between the game logic and the low-level hardware sound interface (`i_sound.c`), managing both sound effects (SFX) and music playback. This file handles channel allocation, 3D positional audio attenuation, stereo panning, pitch variation, and music lifecycle management.

The sound system supports a configurable number of simultaneous sound channels. Each channel tracks which sound effect is playing, which game object originated the sound, and the handle returned by the low-level sound driver. Distance and angle from the listener (the console player's map object) are used each tick to update volume and stereo separation for non-local sounds.

Music is managed separately: only one music track plays at a time, and it is loaded from the WAD as MUS format data, registered with the sound driver, and played with optional looping.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `int` | `snd_SfxVolume` | Current SFX master volume (0-127). Initialized to 15. Adjustable via menu. |
| `int` | `snd_MusicVolume` | Current music master volume (0-127). Initialized to 15. |
| `int` | `numChannels` | Number of simultaneous sound channels. Set by the defaults/config system in `M_misc`. |
| `const char` | `snd_prefixen[]` | Array of prefix characters for sound effect names by category. Purpose is noted as unclear in comments. |

### Static (File-Scope) Variables

| Type | Name | Description |
|------|------|-------------|
| `channel_t*` | `channels` | Dynamically allocated array of `numChannels` channel slots. Allocated in zone memory during `S_Init`. |
| `boolean` | `mus_paused` | Whether music is currently paused (e.g., during game PAUSE). |
| `musicinfo_t*` | `mus_playing` | Pointer to the `musicinfo_t` entry for the currently playing music track, or NULL. |
| `int` | `nextcleanup` | Gametic threshold for triggering sound cache cleanup. Currently unused (cleanup code is commented out). |

### Externally Referenced Variables (Declared extern)

| Type | Name | Source |
|------|------|--------|
| `int` | `snd_MusicDevice` | Sound module — index of music hardware device |
| `int` | `snd_SfxDevice` | Sound module — index of SFX hardware device |
| `int` | `snd_DesiredMusicDevice` | Config — desired music device |
| `int` | `snd_DesiredSfxDevice` | Config — desired SFX device |

---

## Data Structures

### `channel_t` (local to this file)

```c
typedef struct
{
    sfxinfo_t*  sfxinfo;   // Pointer to sound info; NULL means channel is free
    void*       origin;    // Pointer to the map object (mobj_t) that originated the sound
    int         handle;    // Handle returned by I_StartSound; used to query/stop/update
} channel_t;
```

This structure represents one slot in the mixing pool. When `sfxinfo` is NULL the channel is available. The `origin` pointer is used to match sounds to their source objects and to perform distance-based volume culling.

---

## Constants (Defines)

| Name | Value | Description |
|------|-------|-------------|
| `S_MAX_VOLUME` | 127 | Maximum sound volume value |
| `S_CLIPPING_DIST` | `1200*0x10000` | Fixed-point distance beyond which sounds are clipped (inaudible) |
| `S_CLOSE_DIST` | `160*0x10000` | Fixed-point distance at which sounds play at full volume |
| `S_ATTENUATOR` | `(S_CLIPPING_DIST - S_CLOSE_DIST) >> FRACBITS` | Denominator for linear volume attenuation calculation |
| `NORM_PITCH` | 128 | Default pitch value (center/unshifted) |
| `NORM_PRIORITY` | 64 | Default priority for sounds without a link |
| `NORM_SEP` | 128 | Default stereo separation (centered) |
| `S_PITCH_PERTURB` | 1 | Enable pitch perturbation flag |
| `S_STEREO_SWING` | `96*0x10000` | Fixed-point amplitude of stereo panning swing |
| `S_IFRACVOL` | 30 | Percent attenuation from front to back (unused in current code path) |
| `S_NUMCHANNELS` | 2 | Default fallback channel count (superseded by `numChannels`) |

---

## Functions

### `S_Init`

```c
void S_Init(int sfxVolume, int musicVolume)
```

**Purpose:** Initializes the entire sound system at engine startup.

**Parameters:**
- `sfxVolume` — Initial SFX volume level (0-127).
- `musicVolume` — Initial music volume level (0-127).

**Returns:** Nothing.

**Key Logic:**
1. Calls `I_SetChannels()` to initialize the low-level channel mixer (described as a dummy on Linux).
2. Calls `S_SetSfxVolume()` and `S_SetMusicVolume()` to apply initial volumes.
3. Allocates the `channels` array from zone memory using `Z_Malloc` with `PU_STATIC` so it persists for the entire session.
4. Sets all channel `sfxinfo` pointers to NULL (marking all channels free).
5. Marks `mus_paused = 0`.
6. Iterates over all entries in `S_sfx[]` (from index 1 to `NUMSFX`) and sets `lumpnum` and `usefulness` to -1 to indicate uncached sounds.

---

### `S_Start`

```c
void S_Start(void)
```

**Purpose:** Per-level startup. Stops all currently playing sounds and starts the appropriate background music for the new level.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:**
1. Iterates all channels and calls `S_StopChannel` on any that have an active `sfxinfo`.
2. Resets `mus_paused = 0`.
3. Determines the music track number (`mnum`) based on `gamemode`, `gameepisode`, and `gamemap`. For Doom 1 episodes 1-3, music is indexed as `mus_e1m1 + (episode-1)*9 + (map-1)`. Episode 4 maps use a hardcoded lookup into E3 and E2 music (`spmus[]`). For commercial (Doom II) mode, uses `mus_runnin + gamemap - 1`.
4. Calls `S_ChangeMusic(mnum, true)` to start the music looping.
5. Sets `nextcleanup = 15` for the (currently disabled) cache cleanup timer.

---

### `S_StartSoundAtVolume`

```c
void S_StartSoundAtVolume(void* origin_p, int sfx_id, int volume)
```

**Purpose:** The core SFX playback function. Starts playing a sound effect at an explicit volume from an optional originating map object.

**Parameters:**
- `origin_p` — Pointer to the `mobj_t` that is the sound source. May be NULL for UI/global sounds.
- `sfx_id` — Index into `S_sfx[]` identifying the sound effect (a value from `sfxenum_t`).
- `volume` — Starting volume (0-127), before distance attenuation.

**Returns:** Nothing.

**Key Logic:**
1. Validates `sfx_id` range; calls `I_Error` if out of bounds.
2. If `sfx->link` is set, the sound is a linked alias: pitch and priority come from the link, and volume is offset by `sfx->volume`.
3. If `origin` is non-NULL and is not the console player's own map object, calls `S_AdjustSoundParams` to calculate distance-attenuated volume, stereo separation, and pitch. If the return value is 0 (inaudible), returns early.
4. For sounds at the exact same position as the listener, forces `sep = NORM_SEP` (centered).
5. Applies random pitch variation: chainsaw sounds get a small ±8 variance; most other sounds get ±16 variance. `sfx_itemup` and `sfx_tink` are exempt.
6. Calls `S_StopSound(origin)` to evict any prior sound from the same origin.
7. Calls `S_getChannel` to find or allocate a channel slot.
8. Loads the WAD lump number via `I_GetSfxLumpNum` if not already cached.
9. Increments `sfx->usefulness` to track usage for cache eviction.
10. Calls `I_StartSound` to hand the sound to the low-level driver, storing the returned handle in `channels[cnum].handle`.

---

### `S_StartSound`

```c
void S_StartSound(void* origin, int sfx_id)
```

**Purpose:** Convenience wrapper around `S_StartSoundAtVolume` that uses the current global SFX volume.

**Parameters:**
- `origin` — Sound source map object pointer.
- `sfx_id` — Sound effect identifier.

**Returns:** Nothing.

**Key Logic:** Calls `S_StartSoundAtVolume(origin, sfx_id, snd_SfxVolume)`. Contains a large `#ifdef SAWDEBUG` block for debugging chainsaw sound origin tracking, compiled out in normal builds.

---

### `S_StopSound`

```c
void S_StopSound(void* origin)
```

**Purpose:** Stops any sound currently playing from a given origin object.

**Parameters:**
- `origin` — The map object whose sound should be stopped.

**Returns:** Nothing.

**Key Logic:** Scans all channels. If a channel's `origin` matches and `sfxinfo` is non-NULL, calls `S_StopChannel` on it and breaks.

---

### `S_PauseSound`

```c
void S_PauseSound(void)
```

**Purpose:** Pauses the currently playing music track (called when the game is paused).

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** If `mus_playing` is set and music is not already paused, calls `I_PauseSong` and sets `mus_paused = true`.

---

### `S_ResumeSound`

```c
void S_ResumeSound(void)
```

**Purpose:** Resumes a paused music track.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** If `mus_playing` is set and `mus_paused` is true, calls `I_ResumeSong` and clears `mus_paused`.

---

### `S_UpdateSounds`

```c
void S_UpdateSounds(void* listener_p)
```

**Purpose:** Called every game tick to update the volume, stereo separation, and pitch of all actively playing sounds relative to the listener's current position, and to free channels whose sounds have finished.

**Parameters:**
- `listener_p` — Pointer to the console player's `mobj_t`.

**Returns:** Nothing.

**Key Logic:**
1. There is a commented-out section for periodic cache cleanup of unused sound data (DOS 8-bit remnant).
2. Iterates all channels. For each channel with an active `sfxinfo`:
   - If `I_SoundIsPlaying(c->handle)` returns true: recalculates volume/pitch/sep from base values, applies link offsets if any, and if the sound's origin differs from the listener calls `S_AdjustSoundParams`. If inaudible, calls `S_StopChannel`; otherwise calls `I_UpdateSoundParams`.
   - If the sound has stopped playing on the hardware, calls `S_StopChannel` to free the slot.

---

### `S_SetMusicVolume`

```c
void S_SetMusicVolume(int volume)
```

**Purpose:** Sets the music playback volume.

**Parameters:**
- `volume` — New volume (0-127). Values outside this range cause `I_Error`.

**Returns:** Nothing.

**Key Logic:** Calls `I_SetMusicVolume(127)` followed by `I_SetMusicVolume(volume)` — the double call is likely a quirk of the DMX sound library reset behavior. Stores the new value in `snd_MusicVolume`.

---

### `S_SetSfxVolume`

```c
void S_SetSfxVolume(int volume)
```

**Purpose:** Sets the SFX master volume.

**Parameters:**
- `volume` — New volume (0-127).

**Returns:** Nothing.

**Key Logic:** Validates range, stores in `snd_SfxVolume`.

---

### `S_StartMusic`

```c
void S_StartMusic(int m_id)
```

**Purpose:** Starts a music track by ID without looping.

**Parameters:**
- `m_id` — Music identifier from `musicenum_t`.

**Returns:** Nothing.

**Key Logic:** Calls `S_ChangeMusic(m_id, false)`.

---

### `S_ChangeMusic`

```c
void S_ChangeMusic(int musicnum, int looping)
```

**Purpose:** Transitions from the currently playing music to a new track.

**Parameters:**
- `musicnum` — Index into `S_music[]` (a `musicenum_t` value).
- `looping` — Non-zero to loop the track.

**Returns:** Nothing.

**Key Logic:**
1. Validates `musicnum` bounds; calls `I_Error` if invalid.
2. If the requested music is already playing, returns immediately (no restart).
3. Calls `S_StopMusic()` to cleanly shut down the previous track.
4. Looks up the WAD lump for the music if not already cached. Music lump names are prefixed with `"d_"` (e.g., `"d_e1m1"`).
5. Caches the lump data with `W_CacheLumpNum(..., PU_MUSIC)`.
6. Calls `I_RegisterSong` to hand data to the driver, then `I_PlaySong` to start playback.
7. Sets `mus_playing` to the new `musicinfo_t` pointer.

---

### `S_StopMusic`

```c
void S_StopMusic(void)
```

**Purpose:** Stops and unregisters the currently playing music track, releasing its memory.

**Parameters:** None.

**Returns:** Nothing.

**Key Logic:** If `mus_paused`, resumes first (so the driver can properly stop). Calls `I_StopSong`, `I_UnRegisterSong`, then demotes the cached lump to `PU_CACHE` so zone memory can reclaim it. Clears `mus_playing`.

---

### `S_StopChannel`

```c
void S_StopChannel(int cnum)
```

**Purpose:** Stops the sound playing on a specific channel and marks the channel as free.

**Parameters:**
- `cnum` — Channel index (0 to `numChannels-1`).

**Returns:** Nothing.

**Key Logic:**
1. If `I_SoundIsPlaying` reports the sound is still active, calls `I_StopSound`.
2. Checks whether any _other_ channel is playing the same `sfxinfo`. (The `break` statement means it stops checking but does not take different action — a minor code oddity.)
3. Decrements `sfxinfo->usefulness` regardless.
4. Sets `c->sfxinfo = 0` to mark the channel free.

---

### `S_AdjustSoundParams`

```c
int S_AdjustSoundParams(mobj_t* listener, mobj_t* source,
                         int* vol, int* sep, int* pitch)
```

**Purpose:** Computes adjusted volume, stereo separation, and pitch for a non-local sound based on distance and angle from the listener.

**Parameters:**
- `listener` — The listening map object (the console player).
- `source` — The sound-emitting map object.
- `vol` — Pointer to the volume value to be modified in place.
- `sep` — Pointer to the stereo separation value (0=left, 255=right, 128=center) to be modified.
- `pitch` — Pointer to the pitch value to be modified (currently not altered in this function).

**Returns:** 1 if audible (volume > 0), 0 if the sound should be culled.

**Key Logic:**
1. Computes approximate Euclidean distance using the fast formula: `adx + ady - min(adx,ady)/2` (from the Graphics Gems book, p.428). This avoids a square root.
2. If not on map 8 (a special boss map with no distance clipping) and distance exceeds `S_CLIPPING_DIST`, returns 0.
3. Computes the angle from listener to source using `R_PointToAngle2`, then subtracts the listener's own facing angle to get a relative angle.
4. Stereo separation: `*sep = 128 - (FixedMul(S_STEREO_SWING, finesine[angle]) >> FRACBITS)`. The fine sine of the relative angle gives the lateral offset.
5. Volume:
   - If closer than `S_CLOSE_DIST`: full volume.
   - Map 8: special clamped linear falloff formula with minimum of 15.
   - Otherwise: `vol = snd_SfxVolume * (S_CLIPPING_DIST - dist) / S_ATTENUATOR`.

---

### `S_getChannel`

```c
int S_getChannel(void* origin, sfxinfo_t* sfxinfo)
```

**Purpose:** Finds a free or evictable channel slot for a new sound.

**Parameters:**
- `origin` — The sound's origin map object.
- `sfxinfo` — The sound effect info being requested.

**Returns:** Channel index (0 to `numChannels-1`), or -1 if no channel is available.

**Key Logic:**
1. First pass: searches for either a free channel (`sfxinfo == NULL`) or a channel already used by the same `origin` (which gets stopped to be reused).
2. If all channels are occupied, second pass: searches for any channel with a sound of priority >= the new sound's priority. If found, evicts it via `S_StopChannel`.
3. If still no slot, returns -1 (sound is dropped).
4. Assigns `c->sfxinfo` and `c->origin` to the new sound parameters.

---

## Dependencies

| File | Purpose |
|------|---------|
| `i_system.h` | `I_Error`, `I_AllocLow` |
| `i_sound.h` | Low-level sound driver: `I_SetChannels`, `I_StartSound`, `I_StopSound`, `I_SoundIsPlaying`, `I_UpdateSoundParams`, `I_GetSfxLumpNum`, `I_SetMusicVolume`, `I_PauseSong`, `I_ResumeSong`, `I_StopSong`, `I_RegisterSong`, `I_UnRegisterSong`, `I_PlaySong` |
| `sounds.h` / `sounds.c` | `S_sfx[]`, `S_music[]`, `sfxinfo_t`, `musicinfo_t`, `sfxenum_t`, `musicenum_t` |
| `s_sound.h` | Own header; public interface declarations |
| `z_zone.h` | `Z_Malloc`, `Z_ChangeTag` — zone memory allocation |
| `m_random.h` | `M_Random()` — for pitch perturbation |
| `w_wad.h` | `W_GetNumForName`, `W_CacheLumpNum` — WAD lump loading |
| `doomdef.h` | `TICRATE`, `boolean`, game mode constants |
| `p_local.h` | `mobj_t` definition, `players[]` array, `consoleplayer` |
| `doomstat.h` | `gamemode`, `gameepisode`, `gamemap`, `netgame`, etc. |
| `r_main.h` (via p_local) | `R_PointToAngle2` for stereo angle calculation |
| `tables.h` (via r_local) | `finesine[]` for stereo separation computation |
