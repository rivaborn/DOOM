# File Overview

`sounds.h` is the central header for DOOM's audio data definitions. It declares the structures used to describe sound effects (`sfxinfo_t`) and music tracks (`musicinfo_t`), and provides two large enumerations — `sfxenum_t` and `musicenum_t` — that assign symbolic names to every sound effect and music track in the game.

This header is included by nearly every subsystem that needs to reference sounds by name. The game logic, player code, monster AI, status bar, intermission screen, and menu all use the `sfxenum_t` and `musicenum_t` values as arguments to `S_StartSound` and `S_ChangeMusic`. The actual data initialized from these declarations lives in `sounds.c`.

---

## Global Variables

```c
extern sfxinfo_t   S_sfx[];    // Array of all sound effect descriptors
extern musicinfo_t S_music[];  // Array of all music track descriptors
```

Both arrays are defined in `sounds.c`. They are indexed using values from the enumerations below.

---

## Data Structures

### `sfxinfo_t` (`sfxinfo_struct`)

Describes one sound effect entry.

```c
typedef struct sfxinfo_struct sfxinfo_t;

struct sfxinfo_struct
{
    char*        name;         // Up to 6-character WAD lump name base (lowercase, no "ds" prefix)
    int          singularity;  // If true, only one instance of this sound plays at a time
    int          priority;     // Higher = more important; used when evicting channels
    sfxinfo_t*   link;         // If non-NULL, this sound is an alias for another; use linked sound's data
    int          pitch;        // Pitch offset (relative to NORM_PITCH=128) when played via link
    int          volume;       // Volume offset when played via link (added to caller's volume)
    void*        data;         // Pointer to loaded PCM audio data; NULL until cached
    int          usefulness;   // Cache reference count: -1=uncached, 0=eviction candidate, >0=in use
    int          lumpnum;      // WAD lump number; -1 until resolved by I_GetSfxLumpNum
};
```

**Field Details:**
- `singularity`: When true, `s_sound.c` ensures only one channel plays this sound at a time. Used for sounds like `sfx_itemup` and `sfx_wpnup` where overlap would sound wrong.
- `link`: The linked-sound mechanism allows one sound to be played as if it were another, but with pitch/volume modifications. Used by the chaingun (`sfx_chgun`) to reuse the pistol sample at a higher pitch.
- `usefulness`: A reference counting scheme for cache management. The cleanup code in `S_UpdateSounds` was designed to decrement this each cleanup cycle and evict sounds at -1, but this code path is commented out in the Linux release.

---

### `musicinfo_t`

Describes one music track entry.

```c
typedef struct
{
    char*  name;      // Up to 6-character WAD lump base name (WAD lump is "d_" + name)
    int    lumpnum;   // WAD lump number; 0 until resolved by S_ChangeMusic
    void*  data;      // Pointer to loaded MUS format music data; NULL until cached
    int    handle;    // Driver handle returned by I_RegisterSong; 0 until registered
} musicinfo_t;
```

**Field Details:**
- `name`: The base name without the `d_` prefix. For example, Episode 1 Map 1 music is stored as WAD lump `"D_E1M1"` but `name` is `"e1m1"`. `S_ChangeMusic` constructs the full lump name.
- `lumpnum`: Cached after first lookup so repeated calls to `S_ChangeMusic` for the same track avoid redundant name searches.
- `handle`: The integer token returned by `I_RegisterSong`. All subsequent operations on the track (play, stop, pause) use this handle.

---

## Enumerations

### `musicenum_t`

Enumerates all music tracks in the game. Used as the `music_id` parameter in `S_ChangeMusic` and `S_StartMusic`.

```c
typedef enum {
    mus_None,       // Index 0: no music / invalid
    mus_e1m1,       // Episode 1, Map 1
    mus_e1m2,
    // ... e1m3 through e1m9 ...
    mus_e2m1,       // Episode 2, Map 1
    // ... e2m2 through e3m9 ...
    mus_inter,      // Episode intermission screen
    mus_intro,      // Title screen
    mus_bunny,      // End-of-Episode 3 bunny screen
    mus_victor,     // Victory screen
    mus_introa,     // Alternate intro
    // DOOM II tracks (commercial mode):
    mus_runnin,     // MAP01
    mus_stalks,     // MAP02
    mus_countd,     // MAP03
    mus_betwee,     // MAP04
    mus_doom,       // MAP05
    // ... (24 more DOOM II map tracks) ...
    mus_dm2ttl,     // DOOM II title
    mus_dm2int,     // DOOM II intermission
    NUMMUSIC        // Total count
} musicenum_t;
```

Total: 64 entries (index 0 is a null/none sentinel, `NUMMUSIC` is 64).

DOOM 1 uses the first 33 entries (episodes 1-3 plus special screens); DOOM II uses entries 33-63.

---

### `sfxenum_t`

Enumerates all sound effects in the game. Used as the `sound_id` parameter in `S_StartSound` and `S_StartSoundAtVolume`.

```c
typedef enum {
    sfx_None,       // Index 0: dummy
    sfx_pistol,     // Pistol fire
    sfx_shotgn,     // Shotgun fire
    sfx_sgcock,     // Shotgun cock
    sfx_dshtgn,     // Double shotgun fire
    sfx_dbopn,      // Double shotgun open
    sfx_dbcls,      // Double shotgun close
    sfx_dbload,     // Double shotgun load
    sfx_plasma,     // Plasma rifle fire
    sfx_bfg,        // BFG fire
    sfx_sawup,      // Chainsaw startup
    sfx_sawidl,     // Chainsaw idle
    sfx_sawful,     // Chainsaw full (running)
    sfx_sawhit,     // Chainsaw hit
    sfx_rlaunc,     // Rocket launcher fire
    sfx_rxplod,     // Rocket explosion
    sfx_firsht,     // Imp fireball shot
    sfx_firxpl,     // Imp fireball explosion
    sfx_pstart,     // Platform start
    sfx_pstop,      // Platform stop
    sfx_doropn,     // Door open
    sfx_dorcls,     // Door close
    sfx_stnmov,     // Stone/sector move
    sfx_swtchn,     // Switch activate
    sfx_swtchx,     // Switch deactivate
    sfx_plpain,     // Player pain
    sfx_dmpain,     // Demon pain
    sfx_popain,     // (other monster pain)
    // ... monster sight, attack, death sounds ...
    sfx_posit1,     // Zombie sight 1
    sfx_posit2,     // Zombie sight 2
    sfx_posit3,     // Zombie sight 3
    // ... many more ...
    sfx_chgun,      // Chaingun (linked to pistol)
    sfx_tink,       // Tink (item respawn)
    // ... boss/special sounds ...
    sfx_radio,      // Multiplayer radio click
    NUMSFX          // Total count (110)
} sfxenum_t;
```

Total: 110 entries (index 0 is a null sentinel, `NUMSFX` is 110).

---

## Dependencies

| File | Purpose |
|------|---------|
| (none beyond standard C types) | All types used are self-defined in this header |
