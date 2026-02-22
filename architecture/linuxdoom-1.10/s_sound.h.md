# File Overview

`s_sound.h` is the public header for the platform-independent sound manager (`s_sound.c`). It declares the interface used by all other engine subsystems to start, stop, update, and control sound effects and music. Game modules such as the player code, enemy AI, menu system, and status bar include this header to trigger sounds without needing to know anything about the underlying sound hardware or channel management.

The header is intentionally minimal: it exposes only the high-level API and deliberately hides the internal channel structures, volume attenuation logic, and driver calls that live in `s_sound.c`.

---

## Global Variables

This header declares no global variables directly. The volume globals (`snd_SfxVolume`, `snd_MusicVolume`, `numChannels`) are defined in `s_sound.c` and not exposed here; they are accessed only via the setter functions below.

---

## Functions

### `S_Init`

```c
void S_Init(int sfxVolume, int musicVolume);
```

**Purpose:** Initializes the sound system at engine startup. Allocates the channel buffer, sets initial volumes, and marks all sounds as uncached.

**Parameters:**
- `sfxVolume` — Initial SFX master volume (0-127).
- `musicVolume` — Initial music master volume (0-127).

**Returns:** Nothing.

---

### `S_Start`

```c
void S_Start(void);
```

**Purpose:** Per-level sound startup. Stops all currently playing sounds and starts the level's background music track.

**Parameters:** None.

**Returns:** Nothing.

---

### `S_StartSound`

```c
void S_StartSound(void* origin, int sound_id);
```

**Purpose:** Starts playing a sound effect from a given world object at the current master SFX volume. This is the primary interface used by game logic to trigger sounds.

**Parameters:**
- `origin` — Pointer to the `mobj_t` that is the sound source. Pass NULL for UI or positionally global sounds.
- `sound_id` — A value from the `sfxenum_t` enumeration (defined in `sounds.h`).

**Returns:** Nothing.

---

### `S_StartSoundAtVolume`

```c
void S_StartSoundAtVolume(void* origin, int sound_id, int volume);
```

**Purpose:** Starts a sound effect at an explicit volume level, bypassing the master volume for the initial value (which still gets distance-attenuated).

**Parameters:**
- `origin` — Sound source map object pointer.
- `sound_id` — Sound effect identifier from `sfxenum_t`.
- `volume` — Explicit starting volume (0-127).

**Returns:** Nothing.

---

### `S_StopSound`

```c
void S_StopSound(void* origin);
```

**Purpose:** Immediately stops any sound currently playing from the given origin object.

**Parameters:**
- `origin` — The map object whose associated sound should be halted.

**Returns:** Nothing.

---

### `S_StartMusic`

```c
void S_StartMusic(int music_id);
```

**Purpose:** Starts a music track by its numeric identifier without looping.

**Parameters:**
- `music_id` — A value from the `musicenum_t` enumeration (defined in `sounds.h`).

**Returns:** Nothing.

---

### `S_ChangeMusic`

```c
void S_ChangeMusic(int music_id, int looping);
```

**Purpose:** Transitions to a new music track with optional looping. If the requested track is already playing, this is a no-op.

**Parameters:**
- `music_id` — Music identifier from `musicenum_t`.
- `looping` — Non-zero to loop the track indefinitely; zero to play once.

**Returns:** Nothing.

---

### `S_StopMusic`

```c
void S_StopMusic(void);
```

**Purpose:** Stops and unregisters the currently playing music track and releases its cached WAD data.

**Parameters:** None.

**Returns:** Nothing.

---

### `S_PauseSound`

```c
void S_PauseSound(void);
```

**Purpose:** Pauses music playback. Called when the game enters the PAUSE state.

**Parameters:** None.

**Returns:** Nothing.

---

### `S_ResumeSound`

```c
void S_ResumeSound(void);
```

**Purpose:** Resumes previously paused music playback. Called when leaving the PAUSE state.

**Parameters:** None.

**Returns:** Nothing.

---

### `S_UpdateSounds`

```c
void S_UpdateSounds(void* listener);
```

**Purpose:** Called each game tick to update all active sound channels. Recalculates volume and stereo separation relative to the listener's current position, stops sounds that have finished, and culls sounds that are now out of range.

**Parameters:**
- `listener` — Pointer to the console player's `mobj_t`.

**Returns:** Nothing.

---

### `S_SetMusicVolume`

```c
void S_SetMusicVolume(int volume);
```

**Purpose:** Sets the music master volume. Called from the menu system when the player adjusts the music volume slider.

**Parameters:**
- `volume` — New volume level (0-127). Values outside this range cause `I_Error`.

**Returns:** Nothing.

---

### `S_SetSfxVolume`

```c
void S_SetSfxVolume(int volume);
```

**Purpose:** Sets the SFX master volume. Called from the menu system.

**Parameters:**
- `volume` — New volume level (0-127). Values outside this range cause `I_Error`.

**Returns:** Nothing.

---

## Data Structures

None defined in this header. The internal `channel_t` structure is private to `s_sound.c`. The `sfxinfo_t` and `musicinfo_t` structures used by this module are defined in `sounds.h`.

---

## Dependencies

| File | Purpose |
|------|---------|
| `sounds.h` | `sfxenum_t`, `musicenum_t` enumerations used to specify sound/music IDs |
