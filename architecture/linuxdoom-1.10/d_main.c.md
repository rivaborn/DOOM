# File Overview

`d_main.c` contains the DOOM engine entry point (`D_DoomMain`) and the main game loop (`D_DoomLoop`). It is responsible for:

1. **Startup**: parsing command-line arguments, detecting which IWAD is present (shareware, registered, retail, commercial), loading WAD files, and sequentially initialising every subsystem.
2. **Main loop**: running at 35 Hz, processing input events, advancing game logic by one or more tics, rendering the current frame, and submitting audio.
3. **Event dispatch**: maintaining the circular event queue and routing events through the responder chain.
4. **Demo sequencing**: cycling through title screen, demo playback, and credit screen.
5. **Display**: compositing the current game state (3D view, automap, intermission, finale, demo screen) and handling screen-wipe transitions between states.

---

## Global Variables

| Type | Name | Description |
|------|------|-------------|
| `char*[MAXWADFILES]` | `wadfiles` | Null-terminated list of WAD file paths to load. |
| `boolean` | `devparm` | Set when `-devparm` is passed; enables development features. |
| `boolean` | `nomonsters` | Set when `-nomonsters` is passed; suppresses all monster spawning. |
| `boolean` | `respawnparm` | Set when `-respawn` is passed; causes monsters to respawn. |
| `boolean` | `fastparm` | Set when `-fast` is passed; increases monster projectile speed. |
| `boolean` | `drone` | Drone flag (used in multi-screen setups; not normally operational). |
| `boolean` | `singletics` | Debug flag; forces exactly one tic per frame, disabling adaptive timing. |
| `skill_t` | `startskill` | Skill level selected from the command line. |
| `int` | `startepisode` | Episode number selected from the command line. |
| `int` | `startmap` | Map number selected from the command line. |
| `boolean` | `autostart` | True if a map/skill was specified on the command line. |
| `FILE*` | `debugfile` | File pointer for network debug output (opened if `-debugfile` is passed). |
| `boolean` | `advancedemo` | Flag set by `D_AdvanceDemo`; consumed by `D_DoAdvanceDemo` on the next tic. |
| `char` | `wadfile[1024]` | Primary WAD file path (legacy). |
| `char` | `mapdir[1024]` | Development map directory path. |
| `char` | `basedefault[1024]` | Path to the config file (`default.cfg` or `.doomrc`). |
| `event_t[MAXEVENTS]` | `events` | Circular input event buffer (64 slots). |
| `int` | `eventhead` | Next write position in the event buffer. |
| `int` | `eventtail` | Next read position in the event buffer. |
| `gamestate_t` | `wipegamestate` | Tracks the last rendered game state; when it differs from the current state, a wipe is triggered. |
| `int` | `demosequence` | Current position in the title/demo cycle (-1 starts at the beginning). |
| `int` | `pagetic` | Countdown timer for title-screen and credit-screen display duration. |
| `char*` | `pagename` | Name of the WAD lump to display as the current title/credit page. |
| `char` | `title[128]` | Formatted startup title string printed to stdout. |

---

## Functions

### `D_PostEvent`
```c
void D_PostEvent(event_t* ev)
```
Called by platform I/O code to inject an input event into the circular event buffer.
- **ev**: The event to enqueue. Copied by value into `events[eventhead]`.
- Advances `eventhead` with wrap-around using `& (MAXEVENTS-1)`.

---

### `D_ProcessEvents`
```c
void D_ProcessEvents(void)
```
Drains the event queue, passing each event through the responder chain: first `M_Responder` (menu), then `G_Responder` (game). Events consumed by a responder are not passed further. Also guards against accepting input in store-demo mode (commercial IWAD without `map01`).

---

### `D_Display`
```c
void D_Display(void)
```
Composes and renders the current frame. Handles:
- Detecting game-state changes and initiating the screen-wipe effect (`wipe_StartScreen` / `wipe_EndScreen` / `wipe_ScreenWipe` with `wipe_Melt`).
- Calling the appropriate drawer for the current `gamestate`: `AM_Drawer` or `R_RenderPlayerView` for `GS_LEVEL`, `WI_Drawer` for `GS_INTERMISSION`, `F_Drawer` for `GS_FINALE`, `D_PageDrawer` for `GS_DEMOSCREEN`.
- Drawing the HUD (`HU_Drawer`), status bar (`ST_Drawer`), view border, and pause sprite.
- Drawing the menu overlay (`M_Drawer`) on top of everything.
- Calling `NetUpdate` to flush outgoing network packets.
- Calling `I_FinishUpdate` or running the wipe loop until completion.

---

### `D_DoomLoop`
```c
void D_DoomLoop(void)
```
The infinite main game loop. Never returns. Each iteration:
1. Calls `I_StartFrame` for frame-synchronous I/O.
2. Either runs in single-tic mode (if `singletics`): processes one tic explicitly, or calls `TryRunTics` which may run multiple tics to catch up.
3. Updates positional sound (`S_UpdateSounds`).
4. Calls `D_Display` to render the current frame.
5. Optionally calls `I_UpdateSound` and `I_SubmitSound` depending on compile-time sound backend.

---

### `D_PageTicker`
```c
void D_PageTicker(void)
```
Tic callback for title/credit page display. Decrements `pagetic`; calls `D_AdvanceDemo` when it reaches zero.

---

### `D_PageDrawer`
```c
void D_PageDrawer(void)
```
Draws the current title/credit page by blitting the `pagename` WAD lump as a full-screen patch.

---

### `D_AdvanceDemo`
```c
void D_AdvanceDemo(void)
```
Sets `advancedemo = true` to request a demo sequence advance on the next tic. Safe to call from any context.

---

### `D_DoAdvanceDemo`
```c
void D_DoAdvanceDemo(void)
```
Called from the game loop when `advancedemo` is set. Steps `demosequence` and selects the next attraction:
- Step 0: TITLEPIC + intro music (170 tics or 11 seconds for DOOM 2).
- Step 1: Play `demo1`.
- Step 2: CREDIT screen (200 tics).
- Step 3: Play `demo2`.
- Step 4: Game-mode-dependent (CREDIT, HELP2, or TITLEPIC again).
- Step 5: Play `demo3`.
- Step 6 (retail only): Play `demo4`.

---

### `D_StartTitle`
```c
void D_StartTitle(void)
```
Resets `demosequence` to -1 and calls `D_AdvanceDemo` to start the title loop from the beginning.

---

### `D_AddFile`
```c
void D_AddFile(char* file)
```
Appends a WAD/lump file path to the `wadfiles[]` array. Allocates a copy of the string with `malloc`.
- **file**: Path to append.

---

### `IdentifyVersion`
```c
void IdentifyVersion(void)
```
Determines which IWAD is present by testing file accessibility in order: `doom2f.wad` (French), `doom2.wad`, `plutonia.wad`, `tnt.wad`, `doomu.wad`, `doom.wad`, `doom1.wad`. Sets `gamemode` and adds the found WAD to the file list. Also handles `-shdev`, `-regdev`, `-comdev` development shortcuts. Reads `DOOMWADDIR` and `HOME` environment variables.

---

### `FindResponseFile`
```c
void FindResponseFile(void)
```
Scans `myargv` for arguments beginning with `@`. If found, reads the referenced file, parses space-delimited tokens from it, and substitutes them into `myargv`, expanding the command line. This allows very long command lines to be stored in a response file.

---

### `D_DoomMain`
```c
void D_DoomMain(void)
```
The top-level startup function. Sequence:
1. `FindResponseFile` — expand response files.
2. `IdentifyVersion` — detect IWAD and game mode.
3. Parse flags: `-nomonsters`, `-respawn`, `-fast`, `-devparm`, `-deathmatch`, `-altdeath`.
4. Print version title string.
5. Handle `-cdrom`, `-turbo` (scale movement speed), `-wart`, `-file`, `-playdemo`/`-timedemo`.
6. Set `startskill`, `startepisode`, `startmap` from `-skill`, `-episode`, `-warp`.
7. Print modified-game warning if PWADs are loaded and check for fake registered IWAD.
8. Initialise subsystems in order: `V_Init`, `M_LoadDefaults`, `Z_Init`, `W_InitMultipleFiles`, `M_Init`, `R_Init`, `P_Init`, `I_Init`, `D_CheckNetGame`, `S_Init`, `HU_Init`, `ST_Init`.
9. Handle `-statcopy`, `-record`, `-playdemo`, `-timedemo`, `-loadgame`.
10. Start the game: `G_InitNew` or `D_StartTitle`.
11. Enter `D_DoomLoop` — never returns.

---

## Data Structures

No new data structures defined; uses types from `d_event.h`, `doomdef.h`, and others.

---

## Dependencies

| Module | Usage |
|--------|-------|
| `doomdef.h` | Core constants, type enums. |
| `doomstat.h` | Global state variables. |
| `dstrings.h` | `D_DEVSTR`, `D_CDROM`, `SAVEGAMENAME`. |
| `sounds.h` | `mus_intro`, `mus_dm2ttl` music constants. |
| `z_zone.h` | Memory allocation. |
| `w_wad.h` | WAD loading (`W_InitMultipleFiles`, `W_CacheLumpName`, `W_CheckNumForName`). |
| `s_sound.h` | `S_Init`, `S_StartMusic`, `S_UpdateSounds`. |
| `v_video.h` | `V_Init`, `V_DrawPatch`, `V_DrawPatchDirect`, screen buffers. |
| `f_finale.h` | `F_Drawer`. |
| `f_wipe.h` | `wipe_StartScreen`, `wipe_EndScreen`, `wipe_ScreenWipe`, `wipe_Melt`. |
| `m_argv.h` | `M_CheckParm`, `myargc`, `myargv`. |
| `m_misc.h` | `M_LoadDefaults`, `M_ScreenShot`. |
| `m_menu.h` | `M_Init`, `M_Responder`, `M_Drawer`, `M_Ticker`, `M_StartControlPanel`. |
| `i_system.h` | `I_Init`, `I_Error`, `I_GetTime`. |
| `i_sound.h` | `I_UpdateSound`, `I_SubmitSound`. |
| `i_video.h` | `I_InitGraphics`, `I_StartFrame`, `I_StartTic`, `I_UpdateNoBlit`, `I_FinishUpdate`, `I_SetPalette`. |
| `g_game.h` | `G_Responder`, `G_Ticker`, `G_BuildTiccmd`, `G_InitNew`, `G_RecordDemo`, etc. |
| `hu_stuff.h` | `HU_Init`, `HU_Drawer`, `HU_Erase`. |
| `wi_stuff.h` | `WI_Drawer`. |
| `st_stuff.h` | `ST_Init`, `ST_Drawer`. |
| `am_map.h` | `AM_Drawer`. |
| `p_setup.h` | `P_Init`. |
| `r_local.h` | `R_Init`, `R_RenderPlayerView`, `R_FillBackScreen`, `R_DrawViewBorder`, `R_ExecuteSetViewSize`. |
| `d_main.h` | Own header. |
