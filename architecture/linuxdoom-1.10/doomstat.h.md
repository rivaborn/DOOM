# File Overview

`doomstat.h` is the global game state header — the single file that declares `extern` references to every major shared variable in the DOOM engine. As the file's own description states: "All the global variables that store the internal state. Theoretically speaking, the internal state of the engine should be found by looking at the variables collected here."

In practice, the actual definitions of these variables are spread across multiple `.c` files (`g_game.c`, `doomstat.c`, `d_net.c`, etc.), but this header provides the canonical declaration point so any module can access the current game state by including `doomstat.h`. It is a central coupling point of the engine's architecture.

## Global Variables

All variables are declared `extern` here. The sections below follow the groupings used in the header.

### Command-Line Parameters

| Type | Name | Description |
|------|------|-------------|
| `boolean` | `nomonsters` | True if `-nomonsters` was passed: all non-boss monsters are removed at level load. |
| `boolean` | `respawnparm` | True if `-respawn` was passed: killed monsters respawn (like Nightmare difficulty). |
| `boolean` | `fastparm` | True if `-fast` was passed: monsters move and shoot faster. |
| `boolean` | `devparm` | True if `-devparm` was passed: enables developer mode (screenshot on F1, gamma cycling, etc.). |

### Game Mode

| Type | Name | Description |
|------|------|-------------|
| `GameMode_t` | `gamemode` | Which IWAD is loaded (`shareware`, `registered`, `commercial`, `retail`, `indetermined`). |
| `GameMission_t` | `gamemission` | Mission pack identity (`doom`, `doom2`, `pack_tnt`, `pack_plut`). |
| `boolean` | `modifiedgame` | True if any PWAD has been loaded alongside the IWAD. |

### Language

| Type | Name | Description |
|------|------|-------------|
| `Language_t` | `language` | Active language (`english`, `french`, `german`). |

### Level/Episode Selection

| Type | Name | Description |
|------|------|-------------|
| `skill_t` | `startskill` | The skill level selected for the next game start (default for menu). |
| `int` | `startepisode` | The episode number for the next game start. |
| `int` | `startmap` | The map number for the next game start. |
| `boolean` | `autostart` | True when the game should start automatically (set by network arbitration or command-line). |
| `skill_t` | `gameskill` | The active skill level of the current game. |
| `int` | `gameepisode` | The active episode number of the current game. |
| `int` | `gamemap` | The active map number of the current game. |
| `boolean` | `respawnmonsters` | True when monsters should respawn (Nightmare mode or `-respawn`). |

### Multiplayer Game Flags

| Type | Name | Description |
|------|------|-------------|
| `boolean` | `netgame` | True only if this is a real network game with multiple nodes. |
| `boolean` | `deathmatch` | True if started as a deathmatch game (player-vs-player free-for-all). |

### Sound Parameters

| Type | Name | Description |
|------|------|-------------|
| `int` | `snd_SfxVolume` | Master sound effects volume (0-15). |
| `int` | `snd_MusicVolume` | Master music volume (0-15). |
| `int` | `snd_MusicDevice` | Active music device index. |
| `int` | `snd_SfxDevice` | Active SFX device index. |
| `int` | `snd_DesiredMusicDevice` | Desired music device (from config). |
| `int` | `snd_DesiredSfxDevice` | Desired SFX device (from config). |

### Display/Refresh Status Flags

| Type | Name | Description |
|------|------|-------------|
| `boolean` | `statusbaractive` | True when the status bar at the bottom of the screen is being displayed. |
| `boolean` | `automapactive` | True when the automap overlay is displayed. |
| `boolean` | `menuactive` | True when a menu is overlaid on the game view. |
| `boolean` | `paused` | True when the game is paused. |
| `boolean` | `viewactive` | True when the 3D game view is being rendered. |
| `boolean` | `nodrawers` | True to skip all rendering (timing benchmark mode). |
| `boolean` | `noblit` | True to skip blitting to screen (timing benchmark mode). |
| `int` | `viewwindowx` | X origin of the view window in screen coordinates. |
| `int` | `viewwindowy` | Y origin of the view window. |
| `int` | `viewheight` | Height of the view window in pixels. |
| `int` | `viewwidth` | Width of the view window in pixels. |
| `int` | `scaledviewwidth` | View width scaled to screen coordinates. |
| `int` | `viewangleoffset` | Angle offset for 3-screen display mode (ANG90=left, ANG270=right). |

### Player Identity

| Type | Name | Description |
|------|------|-------------|
| `int` | `consoleplayer` | Index (0-3) of the player taking events on this machine. |
| `int` | `displayplayer` | Index of the player whose view is being rendered (spy mode may differ from consoleplayer). |

### Level Statistics

| Type | Name | Description |
|------|------|-------------|
| `int` | `totalkills` | Total number of killable enemies on the current level (for % display). |
| `int` | `totalitems` | Total number of collectable items on the level. |
| `int` | `totalsecret` | Total number of secret areas on the level. |
| `int` | `levelstarttic` | The `gametic` value at the moment the level was loaded. |
| `int` | `leveltime` | Elapsed game tics since the level began (for par time comparison). |

### Demo Playback/Recording

| Type | Name | Description |
|------|------|-------------|
| `boolean` | `usergame` | True when a human player is in control (false during demos, disables save/end game). |
| `boolean` | `demoplayback` | True while playing back a demo file. |
| `boolean` | `demorecording` | True while recording a demo. |
| `boolean` | `singledemo` | True if the game should quit after playing one demo from the command line. |

### Overall Game State

| Type | Name | Description |
|------|------|-------------|
| `gamestate_t` | `gamestate` | Current high-level engine state: `GS_LEVEL`, `GS_INTERMISSION`, `GS_FINALE`, or `GS_DEMOSCREEN`. |

### Internal Engine Parameters

| Type | Name | Description |
|------|------|-------------|
| `int` | `gametic` | The current game tick counter. Increments by 1 every 1/35th of a second. |
| `player_t[MAXPLAYERS]` | `players` | Array of all player state records. |
| `boolean[MAXPLAYERS]` | `playeringame` | True for each player slot that is actively connected. |
| `mapthing_t[MAX_DM_STARTS]` | `deathmatchstarts` | Array of deathmatch player spawn points (max 10). |
| `mapthing_t*` | `deathmatch_p` | Pointer to the next unused entry in `deathmatchstarts`. |
| `mapthing_t[MAXPLAYERS]` | `playerstarts` | Player spawn points for cooperative mode. |
| `wbstartstruct_t` | `wminfo` | Parameters for the world-map/intermission screen. |
| `int[NUMAMMO]` | `maxammo` | Maximum ammo capacity per type (doubles when backpack is collected). |
| `char[1024]` | `basedefault` | Path to the DOOM configuration file. |
| `FILE*` | `debugfile` | File handle for network debug output (opened with `-net` debugging). |
| `boolean` | `precache` | If true, all graphics are loaded into the zone heap at level load time. |
| `gamestate_t` | `wipegamestate` | Set to -1 to force a screen wipe on the next draw; otherwise matches `gamestate`. |
| `int` | `mouseSensitivity` | Mouse sensitivity setting from config file. |
| `boolean` | `singletics` | Debug flag to disable adaptive timing (one real tick = one game tick). |
| `int` | `bodyqueslot` | Circular queue index for the dead player body objects. |
| `int` | `skyflatnum` | WAD lump index of the dummy sky flat texture (F_SKY1). |

### Networking State

| Type | Name | Description |
|------|------|-------------|
| `doomcom_t*` | `doomcom` | Pointer to the network communication structure (set by `I_InitNetwork`). |
| `doomdata_t*` | `netbuffer` | Pointer into `doomcom->data`; the active network packet buffer. |
| `ticcmd_t[BACKUPTICS]` | `localcmds` | Circular buffer of the local player's built tic commands. |
| `int` | `rndindex` | Current position in the random number table (for consistency checks). |
| `int` | `maketic` | The next tic number for which a local command has not yet been built. |
| `int[MAXNETNODES]` | `nettics` | Highest received tic from each network node. |
| `ticcmd_t[MAXPLAYERS][BACKUPTICS]` | `netcmds` | All received tic commands for all players. |
| `int` | `ticdup` | Tic duplication factor (1 = normal, 2-5 = slower networks). |

## Functions

No functions are declared in this header.

## Data Structures

Aside from the `#define MAX_DM_STARTS 10` constant (defining the deathmatch spawn point array size), no new structures are defined here. All types used come from the included headers.

## Dependencies

| File | Reason |
|------|--------|
| `doomdata.h` | Provides WAD data types (`mapthing_t`, etc.) used in `playerstarts`, `deathmatchstarts`. |
| `d_net.h` | Provides `doomcom_t`, `doomdata_t`, `BACKUPTICS`, `MAXNETNODES` for the networking state variables. |
| `d_player.h` | Provides `player_t`, `wbstartstruct_t`, `MAXPLAYERS` for the player state arrays. |
