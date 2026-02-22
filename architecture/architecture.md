# DOOM Source Code Architecture

## 1. Project Overview

DOOM is the landmark first-person shooter developed by id Software, originally released on December 10, 1993. It was one of the defining games of the 1990s, establishing the technical template for 3D action games through its innovative software renderer, networked multiplayer, and moddable WAD file format.

The source code in this repository is the **Linux DOOM version 1.10** release, made publicly available by id Software on December 23, 1997. The release includes four distinct components:

| Component | Directory | Description |
|-----------|-----------|-------------|
| Main engine | `linuxdoom-1.10/` | The complete DOOM game engine for Linux/X11 |
| IPX network driver | `ipx/` | DOS-based IPX LAN network driver (separate executable) |
| Serial network driver | `sersrc/` | DOS-based serial/modem network driver (separate executable) |
| Sound server | `sndserv/` | Standalone UNIX sound server process |

The version constant is defined in `doomdef.h` as `VERSION = 110` (meaning version 1.10).

### Build System

Each component builds independently with its own Makefile:

- `linuxdoom-1.10/Makefile` — builds `linux/linuxxdoom` using GCC with flags `-DNORMALUNIX -DLINUX`. Links against X11 (`-lXext -lX11 -lnsl -lm`). Object files go into the `linux/` subdirectory.
- `sndserv/Makefile` — builds the standalone sound server binary.
- The `ipx/` and `sersrc/` directories contain DOS-targeted C code intended for a 16-bit DOS compiler (Borland/Watcom); they are not built by the Linux Makefile.

The main engine is compiled as C (or C++ via gcc) with `RANGECHECK` enabled for bounds checking assertions. The compile-time flag `SNDSERV` (enabled by default in `doomdef.h`) causes the engine to use the external sound server process over a pipe rather than direct audio hardware access.

---

## 2. Source Directory Structure

### `linuxdoom-1.10/` — Main Engine

| Prefix | Files | Purpose |
|--------|-------|---------|
| `i_` | `i_main.c`, `i_system.c/h`, `i_video.c/h`, `i_sound.c/h`, `i_net.c/h` | Platform interface layer (I = Implementation/interface). All OS-specific code. |
| `d_` | `d_main.c/h`, `d_net.c/h`, `d_event.h`, `d_items.c/h`, `d_player.h`, `d_think.h`, `d_textur.h`, `d_ticcmd.h` | Top-level game driver and data definitions. |
| `g_` | `g_game.c/h` | Game state machine and tick control. |
| `r_` | `r_main.c/h`, `r_bsp.c/h`, `r_segs.c/h`, `r_plane.c/h`, `r_things.c/h`, `r_draw.c/h`, `r_data.c/h`, `r_sky.c/h`, `r_defs.h`, `r_local.h`, `r_state.h` | Software renderer (R = Refresh). |
| `p_` | `p_setup.c/h`, `p_mobj.c/h`, `p_tick.c/h`, `p_map.c`, `p_maputl.c`, `p_user.c`, `p_enemy.c`, `p_inter.c/h`, `p_spec.c/h`, `p_ceilng.c`, `p_doors.c`, `p_floor.c`, `p_lights.c`, `p_plats.c`, `p_pspr.c/h`, `p_saveg.c/h`, `p_sight.c`, `p_switch.c`, `p_telept.c`, `p_local.h` | Play/physics/game world (P = Play). |
| `m_` | `m_argv.c/h`, `m_bbox.c/h`, `m_cheat.c/h`, `m_fixed.c/h`, `m_menu.c/h`, `m_misc.c/h`, `m_random.c/h`, `m_swap.c/h` | Miscellaneous utilities and menus. |
| `s_` | `s_sound.c/h` | Sound channel management (game-level). |
| `hu_` | `hu_stuff.c/h`, `hu_lib.c/h` | Heads-up display. |
| `st_` | `st_stuff.c/h`, `st_lib.c/h` | Status bar. |
| `am_` | `am_map.c/h` | Automap. |
| `wi_` | `wi_stuff.c/h` | Intermission/level statistics screen. |
| `f_` | `f_finale.c/h`, `f_wipe.c/h` | Episode finales and screen wipe transitions. |
| `v_` | `v_video.c/h` | Video/framebuffer utilities. |
| `w_` | `w_wad.c/h` | WAD file I/O. |
| `z_` | `z_zone.c/h` | Zone memory allocator. |
| Core headers | `doomdef.h`, `doomstat.c/h`, `doomdata.h`, `doomtype.h`, `dstrings.c/h`, `d_englsh.h`, `d_french.h` | Fundamental definitions, global state, strings. |
| Tables | `tables.c/h`, `m_fixed.c/h` | Fixed-point math and trigonometric lookup tables. |
| Info | `info.c/h`, `sounds.c/h` | Thing type database and sound definitions. |

### `sndserv/` — Sound Server

| File | Purpose |
|------|---------|
| `soundsrv.c` | Main sound server loop; receives commands from DOOM via pipe, mixes audio |
| `soundsrv.h` | Sound server interface definitions |
| `linux.c` | Linux/voxware audio output |
| `wadread.c/h` | Minimal WAD reader to extract sound lumps |
| `sounds.c/h` | Sound effect definitions (shared with main engine) |
| `soundst.h` | Sound state structure definitions |

### `ipx/` — IPX Network Driver (DOS)

| File | Purpose |
|------|---------|
| `IPXSETUP.C` | IPX network setup and game launch utility |
| `IPXNET.C/H` | IPX packet send/receive via Novell NetWare APIs |
| `DOOMNET.C/H` | Shared DOOMNET interface: TSR setup, interrupt hooking, `doomcom_t` population |
| `IPXSTR.H` | English strings for IPX setup |
| `IPX_FRCH.H` | French strings for IPX setup |

### `sersrc/` — Serial Network Driver (DOS)

| File | Purpose |
|------|---------|
| `SERSETUP.C/H` | Serial/modem connection setup and game launch |
| `PORT.C` | Low-level COM port I/O |
| `DOOMNET.C/H` | Shared DOOMNET interface (same role as ipx/DOOMNET) |
| `SERSTR.H` | English strings |
| `SER_FRCH.H` | French strings |

---

## 3. Core Subsystems

### 3.1 Entry Point and Initialization (`i_main.c`, `d_main.c`)

**`i_main.c`** is the true entry point. It is intentionally trivial:

```c
int main(int argc, char** argv)
{
    myargc = argc;
    myargv = argv;
    D_DoomMain();
    return 0;
}
```

It stores the command-line arguments in the global `myargc`/`myargv` (from `m_argv.c`) and hands control entirely to `D_DoomMain`.

**`D_DoomMain`** in `d_main.c` performs the complete startup sequence:

1. **`FindResponseFile()`** — Scans `argv` for `@filename` arguments and expands them, allowing arbitrarily long command lines stored in a response file.
2. **`IdentifyVersion()`** — Probes the filesystem (via `DOOMWADDIR` env var and `$HOME`) for IWAD files (`doom2.wad`, `doomu.wad`, `doom.wad`, `doom1.wad`, `plutonia.wad`, `tnt.wad`, `doom2f.wad`) to determine the `gamemode` (shareware, registered, retail, commercial). The found WAD is added to the `wadfiles[]` list.
3. **Command-line parsing** — Processes `-nomonsters`, `-respawn`, `-fast`, `-devparm`, `-deathmatch`, `-altdeath`, `-turbo`, `-file`, `-warp`, `-skill`, `-episode`, `-record`, `-playdemo`, `-timedemo`, `-loadgame`.
4. **Subsystem initialization** (in order):
   - `V_Init()` — allocates screen buffers
   - `M_LoadDefaults()` — loads `~/.doomrc` config file
   - `Z_Init()` — initializes the zone memory allocator
   - `W_InitMultipleFiles(wadfiles)` — loads all WAD files
   - `M_Init()` — initializes miscellaneous data (menu system)
   - `R_Init()` — initializes the renderer (textures, sprites, lookup tables)
   - `P_Init()` — initializes play subsystem (switches, specials)
   - `I_Init()` — platform-specific initialization (timer, joystick)
   - `D_CheckNetGame()` — network game setup, populates `doomcom`
   - `S_Init(sfxVolume, musicVolume)` — sound channels
   - `HU_Init()` — heads-up display fonts
   - `ST_Init()` — status bar graphics
5. **Game start** — Selects one of: `G_RecordDemo`, `G_DeferedPlayDemo`, `G_TimeDemo`, `G_LoadGame`, `G_InitNew`, or `D_StartTitle`.
6. **`D_DoomLoop()`** — enters the infinite main loop; never returns.

**`D_DoomLoop`** initializes graphics (`I_InitGraphics`) and enters the main loop:

```c
while (1)
{
    I_StartFrame();       // frame-synchronous I/O
    TryRunTics();         // run one or more game tics
    S_UpdateSounds(...);  // update positional sound
    D_Display();          // render the current frame
    I_UpdateSound();      // mix sound buffer (if not SNDSERV)
    I_SubmitSound();      // send to hardware (if not SNDINTR)
}
```

A `-singletics` debug flag bypasses `TryRunTics` and runs exactly one tic per frame.

**`D_Display`** orchestrates drawing for the current game state:
- `GS_LEVEL` — calls `R_RenderPlayerView`, `HU_Drawer`, `ST_Drawer`, and optionally `AM_Drawer`
- `GS_INTERMISSION` — calls `WI_Drawer`
- `GS_FINALE` — calls `F_Drawer`
- `GS_DEMOSCREEN` — calls `D_PageDrawer` (title/credit screens)

It also manages screen wipe transitions: when `gamestate != wipegamestate`, a wipe is triggered using `wipe_StartScreen`/`wipe_EndScreen`/`wipe_ScreenWipe`.

**Event handling** uses a circular ring buffer (`events[]`, `eventhead`, `eventtail`). Platform code calls `D_PostEvent` to insert events; `D_ProcessEvents` drains the buffer each tic, passing events through the responder chain: `M_Responder` (menu) first, then `G_Responder` (game).

---

### 3.2 Memory Management (`z_zone.c/h`)

DOOM uses a custom **zone memory allocator** that manages a single large contiguous heap obtained from the OS at startup. This design avoids fragmentation and provides a simple cache-eviction mechanism.

**Key structures:**

```c
typedef struct memblock_s
{
    int           size;    // total block size including header
    void**        user;    // pointer to owner's pointer (NULL = free)
    int           tag;     // purge level
    int           id;      // ZONEID (0x1d4a11) for validation
    struct memblock_s* next;
    struct memblock_s* prev;
} memblock_t;

typedef struct
{
    int        size;       // total zone size
    memblock_t blocklist;  // head/tail sentinel
    memblock_t* rover;     // roving allocation pointer
} memzone_t;
```

**Purge tags** (defined in `z_zone.h`):

| Tag | Value | Meaning |
|-----|-------|---------|
| `PU_STATIC` | 1 | Permanent for the entire execution |
| `PU_SOUND` | 2 | Static while sound is playing |
| `PU_MUSIC` | 3 | Static while music is playing |
| `PU_DAVE` | 4 | Miscellaneous static |
| `PU_LEVEL` | 50 | Static until current level exits |
| `PU_LEVSPEC` | 51 | Level-specific thinker |
| `PU_PURGELEVEL` | 100 | Threshold: tags >= 100 are purgeable |
| `PU_CACHE` | 101 | Purgeable cache (WAD lumps etc.) |

**Key functions:**

- **`Z_Init()`** — calls `I_ZoneBase()` to obtain the heap from the OS, initializes a single free block spanning the entire zone.
- **`Z_Malloc(size, tag, ptr)`** — searches from the rover position for a free block of sufficient size. If no free block is found, purgeable blocks (tag >= `PU_PURGELEVEL`) are evicted by nullifying their owner pointer. Blocks are split if the found block is larger than needed. The `ptr` argument is a `void**` to the caller's pointer; the allocator updates it and stores it in the block header, so that if the block is ever evicted, the caller's pointer is automatically set to NULL.
- **`Z_Free(ptr)`** — marks a block free (sets `user = NULL`, `tag = 0`) and coalesces adjacent free blocks.
- **`Z_FreeTags(lowtag, hightag)`** — frees all blocks whose tag falls in the given range. Used at level change to free all `PU_LEVEL` blocks.
- **`Z_ChangeTag(ptr, tag)`** — macro that validates the block ID then calls `Z_ChangeTag2`. Used to change a cached lump from `PU_CACHE` to `PU_STATIC` to pin it in memory.

The rover (a pointer that persists between allocations) implements a **first-fit roving** strategy, distributing allocations across the heap rather than always starting from the beginning. There are never two contiguous free blocks (they are always coalesced).

---

### 3.3 WAD File System (`w_wad.c/h`)

**WAD format overview:**

A WAD file begins with a 12-byte header (`wadinfo_t`):
- `identification[4]` — `"IWAD"` (internal, full game data) or `"PWAD"` (patch, partial override)
- `numlumps` — count of lumps in the directory
- `infotableofs` — byte offset to the lump directory

The lump directory is an array of `filelump_t` entries:
- `filepos` — byte offset of the lump's data within the file
- `size` — byte size of the lump data
- `name[8]` — lump name (null-padded, not necessarily null-terminated, case-insensitive)

**Runtime representation:**

```c
typedef struct
{
    char  name[8];
    int   handle;     // file descriptor
    int   position;   // byte offset in file
    int   size;
} lumpinfo_t;

extern lumpinfo_t* lumpinfo;  // flat array of all lumps across all WADs
extern void**      lumpcache;  // parallel array of cached pointers
extern int         numlumps;
```

**Key functions:**

- **`W_InitMultipleFiles(char** filenames)`** — opens each WAD file, reads its directory, and appends entries to the global `lumpinfo[]` array. Later WADs override lumps with the same name from earlier WADs (enabling PWAD patching).
- **`W_CheckNumForName(name)`** — linear reverse search through `lumpinfo` for a lump by name; returns -1 if not found.
- **`W_GetNumForName(name)`** — same, but calls `I_Error` if not found.
- **`W_CacheLumpNum(lump, tag)`** — the primary access function. Checks `lumpcache[lump]`; if NULL, allocates zone memory with `Z_Malloc(size, tag, &lumpcache[lump])` and reads the lump from disk. Returns the cached pointer. The `&lumpcache[lump]` as the zone user pointer means the cache entry is automatically cleared when the zone evicts the block.
- **`W_CacheLumpName(name, tag)`** — convenience wrapper.
- **`W_ReadLump(lump, dest)`** — unconditional read into caller-supplied buffer (bypasses cache).

The lump cache integrates directly with the zone allocator's eviction mechanism: when zone memory pressure evicts a `PU_CACHE` block, it zeroes `lumpcache[lump]`, so the next access re-reads from disk transparently.

---

### 3.4 Renderer (`r_*.c/h`)

DOOM uses a **BSP-based, column-oriented software renderer** that operates entirely in software on a 320x200 8bpp framebuffer. It exploits the BSP tree precomputed by the map editor to achieve correct front-to-back ordering without a z-buffer.

#### Pipeline overview

```
R_RenderPlayerView
  └─ R_SetupFrame (viewport, view matrices)
  └─ R_ClearClipSegs / R_ClearDrawSegs / R_ClearPlanes / R_ClearSprites
  └─ R_RenderBSPNode (BSP traversal → wall rendering)
  └─ R_DrawPlanes (floor/ceiling spans)
  └─ R_DrawMasked (sprites and masked walls, back-to-front)
```

#### `r_main.c` — Entry point and setup

`R_RenderPlayerView(player_t* player)` is called once per frame. It:
1. Sets `viewx`, `viewy`, `viewz`, `viewangle`, `viewcos`, `viewsin` from the player's position and view bob.
2. Calls `R_SetupFrame` to compute lighting tables and set `extralight` from gun flash.
3. Clears the clip arrays, drawseg list, visplane list, and vissprite list.
4. Calls `R_RenderBSPNode(numnodes-1)` to traverse the BSP tree.
5. Calls `R_DrawPlanes` and `R_DrawMasked`.

Key precomputed tables built during `R_Init`:
- `viewangletox[FINEANGLES/2]` — maps fine angle to screen X column
- `xtoviewangle[SCREENWIDTH+1]` — maps screen X to view angle
- `scalelight[LIGHTLEVELS][MAXLIGHTSCALE]` — light diminishment by scale
- `zlight[LIGHTLEVELS][MAXLIGHTZ]` — light diminishment by distance

The **field of view** is 2048 fine-angle units wide (`FIELDOFVIEW = 2048`).

Column and span draw functions are assigned through function pointers (`colfunc`, `fuzzcolfunc`, `transcolfunc`, `spanfunc`), allowing different renderers to be selected at runtime (e.g., fuzzy/spectre rendering).

#### `r_bsp.c` — BSP traversal

`R_RenderBSPNode(bspnum)` recursively walks the BSP tree. For each node, it:
1. Determines which side of the partition line the viewpoint is on (front/back) using `R_PointOnSide`.
2. Recursively renders the front child first (guaranteeing front-to-back order).
3. Checks whether the back child's bounding box is visible using `R_CheckBBox` against the horizontal clip ranges.
4. Renders the back child if visible.

At leaf nodes (subsectors, indicated by bit `NF_SUBSECTOR = 0x8000` in the child index), `R_Subsector` is called, which processes each seg (wall segment) in the subsector with `R_AddLine`.

`R_AddLine` clips the seg to the view frustum and calls `R_StoreWallRange`, which records it in the `drawsegs[]` array and marks horizontal screen ranges as clipped via `solidsegs[]`. The solid seg list prevents rendering walls behind already-solid walls — this is the core occlusion mechanism.

#### `r_segs.c` — Wall segment rendering

`R_StoreWallRange` computes the scale (distance-based) for each column of a wall segment and generates the actual pixel columns. For two-sided linedefs, it handles upper, middle, and lower texture sections, computing which portions are visible given the floor and ceiling heights of adjacent sectors. Masked textures (translucent mid-textures on two-sided lines) are deferred and stored in the drawseg for later rendering by `R_DrawMasked`.

#### `r_plane.c` — Floor and ceiling rendering

Floors and ceilings are rendered as **visplanes**. A visplane (`visplane_t`) represents a horizontal surface at a particular height, flat texture, and light level, spanning a range of screen columns. Up to `MAXVISPLANES = 128` visplanes can exist per frame — exceeding this is a hard limit. The `top[]` and `bottom[]` arrays (of size `SCREENWIDTH`) store the pixel row extents for each column.

`R_DrawPlanes` iterates all visplanes and calls the `spanfunc` to draw horizontal spans (textured rows). For sky flats, `R_DrawSkyColumn` is called instead. The sky is rendered as a column operation (not a span), using the view angle to look up the sky texture column.

#### `r_things.c` — Sprite rendering

During BSP traversal, `R_AddSprites(sector)` is called for each visible subsector. For each `mobj_t` in the sector's thing list, `R_ProjectSprite` determines whether it is within the view frustum and creates a `vissprite_t` describing the projected sprite. Vissprites are added to a doubly-linked list sorted by scale (distance).

`R_DrawMasked` traverses the vissprite list from back to front (furthest first), calling `R_DrawVisSprite` for each, which draws the sprite column by column, clipping against the silhouettes recorded in `drawsegs[]`.

#### `r_draw.c` — Column and span drawing

All pixel-level drawing passes through routines in `r_draw.c`:
- `R_DrawColumn` — draws a single vertical column of texture (wall/sprite). Iterates pixel rows, stepping through a fixed-point texture coordinate.
- `R_DrawFuzzColumn` — draws the spectre/fuzzy effect by randomizing which row is sampled using a fuzz table.
- `R_DrawTranslatedColumn` — applies a colormap translation for player-colored sprites.
- `R_DrawSpan` — draws a horizontal floor/ceiling span using fixed-point u/v stepping.

All drawing goes to `screens[0]`, a 320x200 byte array. The frame buffer architecture is deliberately isolated — other renderer files only know about coordinates, not the framebuffer layout.

#### `r_data.c` — Texture composition

DOOM textures are **composites** built from multiple patches (sub-images). `R_InitTextures` reads the `TEXTURE1` (and `TEXTURE2` for registered) lumps, which list texture names and the patches that compose them. `R_GenerateLookup` precomputes which patches cover each column, enabling `R_GetColumn` to efficiently return a pointer to a single column of a texture.

Flat textures (floors/ceilings) are 64x64 raw arrays stored as individual lumps between `F_START`/`F_END` markers.

Sprites are patches (`patch_t`) loaded by `R_InitSprites` from lumps named by the sprite naming convention: `NNNNFx` or `NNNNFxFx`, where `NNNN` is the sprite name, `F` is the frame letter, and `x` is the rotation (0 = no rotation, 1-8 = eight rotations). Horizontal flipping (`NNNNFxFx`) saves space.

---

### 3.5 Map/Level System (`p_setup.c`, `p_mobj.c/h`)

#### WAD map lump format

Each map in a WAD file is identified by a label lump (`ExMy` or `MAPxx`) followed by exactly ten data lumps in fixed order:

| Index | Name | Content |
|-------|------|---------|
| 0 | Label | Map name marker (e.g., `E1M1`) |
| 1 | `THINGS` | Spawn locations, types, flags for all map objects |
| 2 | `LINEDEFS` | Wall lines (vertex indices, flags, special, tag, sidedefs) |
| 3 | `SIDEDEFS` | Texture names and offsets for each side of a linedef |
| 4 | `VERTEXES` | 2D coordinates (short integers) |
| 5 | `SEGS` | BSP-split line segments (vertex indices, angle, linedef, side, offset) |
| 6 | `SSECTORS` | BSP leaf nodes: count and index of first seg |
| 7 | `NODES` | BSP tree: partition line, child bounding boxes, child indices |
| 8 | `SECTORS` | Floor/ceiling heights, flat names, light level, special, tag |
| 9 | `REJECT` | Sector-sector LOS rejection table (bit matrix) |
| 10 | `BLOCKMAP` | Grid of 128x128-unit cells listing intersecting linedefs |

`P_SetupLevel` (called from `G_DoLoadLevel`) orchestrates loading all these lumps in sequence, converting the raw WAD binary formats into runtime structures allocated in zone memory with tag `PU_LEVEL`.

#### Map object system (`mobj_t`)

`mobj_t` (map object, "thing") is the central actor type. Every entity in the game — players, monsters, projectiles, pickups, decorations — is an `mobj_t`.

```c
typedef struct mobj_s
{
    thinker_t   thinker;    // must be first: prev/next/function pointers
    fixed_t     x, y, z;   // world position (fixed-point)
    struct mobj_s* snext, *sprev;  // sector thing list
    angle_t     angle;
    spritenum_t sprite;
    int         frame;
    struct mobj_s* bnext, *bprev;  // blockmap list
    struct subsector_s* subsector;
    fixed_t     floorz, ceilingz;
    fixed_t     radius, height;
    fixed_t     momx, momy, momz;  // momentum
    int         validcount;
    mobjtype_t  type;
    mobjinfo_t* info;
    int         tics;          // state timer countdown
    state_t*    state;         // current animation state
    int         flags;         // mobjflag_t bitfield
    int         health;
    int         movedir;       // 0-7 (DI_EAST..DI_SOUTHEAST)
    int         movecount;
    struct mobj_s* target;     // thing being chased or shot at
    int         reactiontime;
    int         threshold;
    struct player_s* player;  // non-NULL only for MT_PLAYER
    int         lastlook;
    mapthing_t  spawnpoint;
    struct mobj_s* tracer;    // for missile trackers
} mobj_t;
```

The `thinker_t` at the head of `mobj_t` means any `mobj_t*` can be cast to `thinker_t*`, making it a participant in the thinker system. The `function` field of the thinker is set to `P_MobjThinker` for live mobjs.

`P_SpawnMobj` creates a new mobj: allocates with `Z_Malloc(sizeof(mobj_t), PU_LEVEL, NULL)`, sets fields from `mobjinfo[type]`, adds it to the thinker list with `P_AddThinker`, and links it into the blockmap and sector list with `P_SetThingPosition`.

`P_RemoveMobj` unlinks from all lists and marks the thinker for lazy deletion.

#### Thinker system (`p_tick.c/h`)

```c
typedef struct thinker_s
{
    struct thinker_s* prev;
    struct thinker_s* next;
    think_t function;        // function pointer union
} thinker_t;
```

All thinkers are linked in a doubly-linked circular list headed by `thinkercap`. `P_RunThinkers` iterates this list each tic, calling `thinker->function.acp1(thinker)` for each active thinker. Thinkers marked for removal (function set to `(actionf_v)(-1)`) are freed and unlinked during traversal.

Besides `mobj_t` instances, thinkers are also used for sector specials (moving floors, ceilings, lights, platforms) — each active special gets its own thinker allocated from `PU_LEVSPEC` zone memory.

#### Blockmap (`p_setup.c`)

The blockmap divides the map into a rectangular grid of 128x128 world-unit cells. Each cell stores a list of `line_t*` pointers for all linedefs that intersect the cell. The `blocklinks[]` array provides per-cell linked lists of `mobj_t*` for all interactive objects in that cell.

Used by collision detection (`P_BlockThingsIterator`, `P_BlockLinesIterator`) to avoid checking every object/line against every other object/line. Objects with `MF_NOBLOCKMAP` do not appear in blocklinks.

#### REJECT table

The REJECT lump is a bit matrix indexed by `[sector1][sector2]`. If the bit is set, the engine skips the detailed `P_CheckSight` calculation and assumes the sectors cannot see each other. This is precomputed by the BSP compiler based on geometry. Without it, enemy AI line-of-sight checks would be much more expensive.

---

### 3.6 Game Logic / Physics (`p_map.c`, `p_maputl.c`, `p_user.c`)

#### Movement and collision (`p_map.c`)

`P_TryMove(mobj, x, y)` attempts to move a thing to a new position:
1. Sets up the trial bounding box `tmbbox[]` around the new position.
2. Calls `P_CheckPosition` which iterates blockmap cells to find all things and linedefs overlapping the bounding box.
3. For each linedef found, checks for blockage (solid one-sided lines, height differences exceeding `MAXSTEP = 24`).
4. For each thing found, checks for overlap and solid collision.
5. If the move is valid (`floatok` is true and floor-to-ceiling gap is sufficient), the thing is actually moved: `P_UnsetThingPosition`, update `x/y`, `P_SetThingPosition`.

Slides along walls are handled by `P_SlideMove`, which recursively attempts moves along wall normals when a direct move is blocked.

**Hitscan weapons** (`P_LineAttack`, `P_AimLineAttack`): these cast a ray from the shooter's position. `P_PathTraverse` walks the ray through the blockmap, calling a callback for each line and thing intersected. `PTR_ShootTraverse` applies damage or spawns bullet puffs. `P_AimLineAttack` uses a similar ray traversal to find an auto-aim target within a vertical slope range.

**Missile movement** is handled by `P_MobjThinker`: each tic applies momentum (`momx`, `momy`), calls `P_TryMove`, and handles explosions on impact.

#### Player movement (`p_user.c`)

`P_MovePlayer(player)` translates the `ticcmd_t` stored in `player->cmd` into actual mobj momentum:
- `forwardmove` and `sidemove` are scaled by the movement speed tables and projected along/perpendicular to `mo->angle`.
- `angleturn` is added directly to `mo->angle`.
- Running (shift held) uses `forwardmove[1]` / `sidemove[1]` vs. the slower `[0]` values.

`P_CalcHeight(player)` computes `player->viewz` (eye height), applying the bob effect based on horizontal momentum magnitude. View height smoothly tracks the floor height when landing.

---

### 3.7 AI / Monster Behavior (`p_enemy.c`)

Monster AI operates through the state machine system. Each tic of a monster's life, `P_MobjThinker` decrements `mobj->tics`; when it reaches zero, it transitions to the next state via `P_SetMobjState`, which may call an action function.

**State machine and action functions:**

Defined in `info.c/info.h`, each `state_t` contains:
- Sprite and frame to display
- Duration in tics
- Next state index
- An optional action function pointer (`actionf_t`)

Action functions (prefixed `A_`) are called as the state is entered. Examples:
- `A_Look` — scans for player targets by checking sound-alertness and calling `P_CheckSight`
- `A_Chase` — pursues the current target: calls `P_Move` to take a step, calls `P_NewChaseDir` to select a new direction if needed, attempts to attack when in range
- `A_FaceTarget` — rotates the monster toward its target
- `A_PosAttack`, `A_TroopAttack`, etc. — specific attack functions per monster type

**Direction selection (`P_NewChaseDir`):**

Uses a simple 8-directional grid navigation. Selects the best primary and diagonal directions toward the target, tries them in order, and falls back to perpendicular directions if blocked. The `movedir` field stores the chosen direction (0-8 using `dirtype_t`: `DI_EAST` through `DI_NODIR`). `movecount` counts down steps before re-evaluating.

**Line-of-sight (`p_sight.c`):**

`P_CheckSight(t1, t2)`:
1. First checks the REJECT table: if the bit for `[t1->subsector->sector][t2->subsector->sector]` is set, returns false immediately.
2. Computes `sightzstart` (eye height of looker) and slope bounds to top and bottom of target.
3. Calls `P_CrossSubsectors` to trace the line of sight through the BSP tree, checking whether any two-sided lines block the vertical slope.

This two-level check (REJECT + detailed) is efficient: most monster pairs are in the REJECT table, and only nearby monsters proceed to the full BSP trace.

**Monster waking:**

Monsters start deaf (`MF_AMBUSH` flag) or awake. Sound propagation uses `P_RecursiveSound` which marks sectors reachable through non-blocking lines as having heard a sound (stored in `sector->soundtraversed` and `sector->soundtarget`). During `A_Look`, a monster checks if its sector's `soundtarget` is the player.

---

### 3.8 Game Sectors / Specials (`p_spec.c`, `p_ceilng.c`, `p_doors.c`, `p_floor.c`, `p_lights.c`, `p_plats.c`)

**Special line triggers** (`p_spec.c`, `p_switch.c`):

Lines with a non-zero `special` field trigger effects when activated. Activation modes:
- `W1`/`WR` — walk-over (one-shot or repeatable): player crosses the line
- `S1`/`SR` — switch (one-shot or repeatable): player uses the line
- `G1`/`GR` — gun (projectile hit)
- `D1`/`DR` — door (special case of switch-activated door)

`P_CrossSpecialLine` (walk trigger), `P_UseSpecialLine` (use trigger), and `P_ShootSpecialLine` (gun trigger) dispatch to specific effect functions based on the `special` number. Effect numbers are defined by the DOOM level format specification (e.g., special 1 = vertical door, special 11 = exit level).

**Moving sectors:**

Each type of moving geometry has its own source file and thinker type:

- **`p_doors.c`** — `vldoor_t` thinker. Doors move the ceiling up/down. Door types include normal (open/close), stay-open, fast, locked (requiring key cards). `T_VerticalDoor` runs each tic, adjusting `sector->ceilingheight`.
- **`p_ceilng.c`** — `ceiling_t` thinker for crusher ceilings, ceiling raises/lowers. `T_MoveCeiling` uses `T_MovePlane` helper.
- **`p_floor.c`** — `floormove_t` thinker. Handles floor raises, lowers, stair builds. `T_MoveFloor`.
- **`p_plats.c`** — `plat_t` thinker for platforms (rising/lowering, perpetual oscillation). `T_PlatRaise`.
- **`p_lights.c`** — various light thinkers (`fireflicker_t`, `lightflash_t`, `strobe_t`, `glow_t`). Modify `sector->lightlevel` each tic to create flickering, strobing, and glowing effects.

All moving sector thinkers call `T_MovePlane` from `p_floor.c`, which performs the actual height change and handles floor/ceiling crush logic (damaging things caught between moving surfaces).

**Teleporters (`p_telept.c`):**

`EV_Teleport` finds the destination sector by tag, locates a `MT_TELEPORTMAN` spawn in that sector, and relocates the thing there. A teleport fog is spawned at source and destination.

**`P_UpdateSpecials`** (called from `P_Ticker`):

- Advances animated wall textures (cycling through texture sequences defined in `SWITCHES` and `ANIMDEFS`).
- Counts down scrolling line timers.
- Handles end-level triggers (special 11) for exit to intermission.

---

### 3.9 Game State Machine (`g_game.c/h`)

`g_game.c` maintains the high-level game state and processes the sequence of game actions each tic.

**Game states** (`gamestate_t` in `doomdef.h`):

| State | Value | Description |
|-------|-------|-------------|
| `GS_LEVEL` | 0 | Playing a level |
| `GS_INTERMISSION` | 1 | Level end statistics |
| `GS_FINALE` | 2 | Episode finale sequence |
| `GS_DEMOSCREEN` | 3 | Title/demo loop |

**Game actions** (`gameaction_t` in `g_game.h`):

Actions are deferred requests processed at the start of the next tic by `G_Ticker`:
- `ga_nothing` — no pending action
- `ga_loadlevel` → `G_DoLoadLevel`
- `ga_newgame` → `G_DoNewGame`
- `ga_loadgame` → `G_DoLoadGame`
- `ga_savegame` → `G_DoSaveGame`
- `ga_playdemo` → `G_DoPlayDemo`
- `ga_completed` → `G_DoCompleted` (triggers intermission)
- `ga_victory` → `G_DoVictory`
- `ga_worlddone` → `G_DoWorldDone` (end intermission)

**`G_Ticker`:**

Called each tic. Processes the pending `gameaction` first. Then, depending on `gamestate`:
- `GS_LEVEL` — calls `P_Ticker` (runs the play simulation) and `HU_Ticker`/`ST_Ticker`/`AM_Ticker`
- `GS_INTERMISSION` — calls `WI_Ticker`
- `GS_FINALE` — calls `F_Ticker`
- `GS_DEMOSCREEN` — calls `D_PageTicker`

**Tic commands (`ticcmd_t`):**

```c
typedef struct
{
    char  forwardmove;   // scaled movement: *2048 = fixed_t speed
    char  sidemove;      // lateral movement
    short angleturn;     // <<16 for angle delta
    short consistancy;   // consistency check value for net games
    byte  chatchar;      // chat message character
    byte  buttons;       // action buttons bitmask
} ticcmd_t;
```

`G_BuildTiccmd` constructs a ticcmd each tic from the current keyboard/mouse/joystick state. It consults `gamekeydown[]`, `mousebuttons[]`, `joybuttons[]`, `mousex`/`mousey`, and `joyxmove`/`joyymove`. The resulting command is stored in `netcmds[consoleplayer][maketic % BACKUPTICS]` for transmission to other nodes.

`G_Ticker` applies ticcmds: for each player in game, `players[i].cmd = netcmds[i][gametic % BACKUPTICS]`, then `P_Ticker` processes all player commands.

**Demo recording/playback:**

Demos are stored as a sequence of `ticcmd_t` structures preceded by a header (version number, skill, episode, map, player flags). `G_RecordDemo` / `G_BeginRecording` / `G_WriteDemoTiccmd` write ticcmds to a buffer on each tic. `G_PlayDemo` / `G_ReadDemoTiccmd` replace live input by reading from the buffer. Since the game is fully deterministic given the same sequence of ticcmds (using a fixed random number table indexed by `P_Random`), demos play back identically.

**Save/load (`p_saveg.c`):**

Saves are raw binary dumps of the game state to files named `doomsav0.dsg` through `doomsav5.dsg` (or `c:\doomdata\doomsav*.dsg` in CD-ROM mode). Save size is `SAVEGAMESIZE = 0x2c000` bytes. `P_SaveGame` serializes: level statistics, world variables, all thinker data (walking the thinker list), sector specials, ceilings, floors, lights, platforms, doors, and player data. `P_UnArchivePlayers`, `P_UnArchiveWorld`, `P_UnArchiveThinkers`, `P_UnArchiveSpecials` restore them, with pointer fixup for thinker cross-references.

---

### 3.10 Sound System (`s_sound.c/h`, `i_sound.c/h`, `sndserv/`)

The sound system is layered into three levels:

#### Game-level sound management (`s_sound.c`)

Manages up to `MAXCHANNELS` concurrent sound channels. Each channel holds:
- The `mobj_t*` origin (or NULL for non-positional sounds)
- The `sfxinfo_t*` definition
- A handle returned by the platform layer

**`S_StartSound(origin, sound_id)`:**
1. Looks up `sfxinfo_t` from `S_sfx[sound_id]`.
2. Checks if the sound is already playing from the same origin; if so, stops it.
3. Finds a free or lowest-priority channel.
4. Computes stereo panning and volume based on distance from the listener.
5. Calls `I_StartSound` (platform layer) to begin playback.

**`S_UpdateSounds(listener)`:** Called each frame. For each active channel with a positional origin, recomputes distance and panning. Channels whose origins have been deleted (pointer cleared) are stopped.

**`S_StartMusic(music_id)` / `S_ChangeMusic(music_id, looping)`:** Delegates to `I_PlaySong` via the registered music callbacks.

#### Platform sound interface (`i_sound.c`)

The Linux implementation in `i_sound.c` uses the Linux OSS (Open Sound System) `/dev/dsp` interface (via `linux/soundcard.h`).

When compiled with `SNDSERV` (the default), `i_sound.c` communicates with the `sndserv` binary via an IPC mechanism (pipes). The main engine sends sound commands and the server handles mixing.

When compiled without `SNDSERV`, `i_sound.c` directly manages the audio device: it decodes DOOM sound format (8-bit unsigned PCM at 11025 Hz with a 4-byte header) and mixes up to 8 channels internally using software mixing into a stereo 16-bit buffer at 11025 Hz.

#### Sound server (`sndserv/soundsrv.c`)

The standalone sound server is a separate UNIX process that:
1. Opens the audio device (`/dev/dsp`).
2. Reads WAD sound lumps via its own minimal WAD reader (`wadread.c`).
3. Receives commands from the main DOOM process (via stdin/pipe).
4. Mixes active sounds into a stereo buffer (`MIXBUFFERSIZE = 512*2*2` bytes).
5. Writes mixed audio to the sound device at 11025 Hz (`SPEED = 11025`).

The sound server processes commands encoded as a stream of bytes: start-sound (with channel, sound ID, volume, separation), stop-sound, and update-sound commands.

---

### 3.11 HUD / Status Bar (`st_stuff.c/h`, `st_lib.c/h`, `hu_stuff.c/h`, `hu_lib.c/h`)

#### Status bar (`st_stuff.c`, `st_lib.c`)

The status bar occupies the bottom 32 pixels of the screen (or is hidden in full-screen mode). It displays:
- Health percentage (red number panel)
- Currently equipped weapon (arm indicator)
- Ammunition count for current weapon
- Armor percentage
- Key cards held
- Face panel (DOOMGUY face, reacts to damage and health level)
- Ammo counts for all four ammo types (small display in lower corners)

`st_lib.c` provides generic widget types used by the status bar:
- `STlib_initNum` / `STlib_updateNum` — multi-digit number display
- `STlib_initPercent` — percentage number with '%' sign
- `STlib_initBinIcon` / `STlib_updateBinIcon` — on/off icon (keys, arms)
- `STlib_initMultIcon` / `STlib_updateMultIcon` — multi-image selector (face, ammo type)

`ST_Ticker` updates all widgets each tic. `ST_Drawer` renders them by blitting patches from WAD lumps. `ST_Responder` handles cheat code detection via the cheat sequence buffer in `m_cheat.c`.

#### Heads-up display (`hu_stuff.c`, `hu_lib.c`)

The HUD overlays text on the game view for:
- Chat messages (multiplayer)
- Player messages (pickup notifications, secret found, etc.)
- Map title (displayed briefly at level start)
- Crosshair (if applicable)

`hu_lib.c` provides text line widgets (`hu_textline_t`) and message widgets (`hu_stext_t`, `hu_itext_t`) that render using the small HU font (found in WAD as `STCFN` patches). `HU_Ticker` manages message timeout. `HU_Responder` handles chat character input.

---

### 3.12 Automap (`am_map.c/h`)

The automap renders an overhead 2D view of explored map geometry. It is toggled by the Tab key.

**Features:**
- Displays explored linedefs (seen by player), color-coded by type (two-sided, one-sided, secret, unrevealed)
- Follows the player's position or allows free panning (arrow keys)
- Zoom in/out
- Optional overlay mode (renders on top of the game view)
- Cheat codes to reveal full map (`IDDT`)
- `MAXMAPWIDTH`/`MAXMAPHEIGHT` constants control drawing scale

The automap uses its own coordinate system scaled from world coordinates to screen space. `AM_drawWalls` iterates all `lines[]` and draws only those with `ML_MAPPED` flag set (or all lines in cheat mode). Player position is indicated by an arrow drawn with `AM_drawLineCharacter`.

---

### 3.13 Menus (`m_menu.c/h`)

The menu system is a simple state machine. `M_Responder` intercepts key events when `menuactive` is true. The menu is structured as a stack-like navigation with a current menu, highlighted item, and item list.

Each menu is a `menu_t` structure containing:
- A title string lump name
- Array of `menuitem_t` (label, activation callback, keyboard shortcut)
- Prev menu for back navigation
- Cursor position

Top-level menus: New Game, Options (sound volume, screen size, mouse sensitivity, message toggle), Load Game, Save Game, Read This (help screens), Quit Game.

`M_Drawer` renders the active menu by drawing the background skull cursors and item text. Input handling advances through menu levels, selects items, and adjusts numeric values (volume, screen size) with left/right.

The options screen uses slider bars for sound/music volume and screen size, drawn by blinking the correct gem graphic in the slider.

---

### 3.14 Intermission / Finale (`wi_stuff.c/h`, `f_finale.c/h`, `f_wipe.c/h`)

#### Intermission screen (`wi_stuff.c`)

After completing a level, the game transitions to `GS_INTERMISSION`. `WI_Start(wbstartstruct_t*)` receives statistics (kill/item/secret counts, time, par time, player frags). The intermission shows:

- For DOOM 1: the episode map with animated pointers showing progress between levels.
- For DOOM 2: a simple tally screen.
- Kill/item/secret percentages tallied upward in a count animation.
- Frag matrix in deathmatch mode.

`WI_Ticker` advances the animation state machine. `WI_Drawer` renders the current state.

#### Episode finale (`f_finale.c`)

After the final level of an episode, `F_StartFinale` runs a text scroll (cast of characters in DOOM 2, text story in DOOM 1) followed by a final graphic. `F_Ticker` advances the text reveal at one character per 3 tics. DOOM 2 also includes the "cast of characters" screen where all monsters parade across the screen with death animations.

#### Screen wipe (`f_wipe.c`)

Screen wipe transitions occur when the game state changes. The implemented wipe is `wipe_Melt`: the new screen slides in from the top with independent column delays (a random pattern giving the "melting" appearance).

- `wipe_StartScreen` — captures the current screen to a backup buffer.
- `wipe_EndScreen` — captures the new screen.
- `wipe_ScreenWipe(wipe_Melt, ...)` — per-frame update that advances each column's melt position using a precomputed random starting delay. Returns true when complete.

---

### 3.15 Networking (`d_net.c/h`, `i_net.c/h`, `ipx/`, `sersrc/`)

#### Deterministic tic-based model

DOOM networking uses a **lockstep synchronization** model: all machines must process the same sequence of tic commands (`ticcmd_t`) in identical order. No game state is transmitted — only input commands. Since the game is fully deterministic given fixed inputs (including via the `P_Random` table), all machines produce identical game states.

The protocol centers on `BACKUPTICS = 12`: a rolling buffer of ticcmds. The engine cannot advance `gametic` until it has received ticcmds from all nodes for that tic.

Key counters in `d_net.c`:
- `gametic` — the tic currently being executed (or about to be)
- `maketic` — the tic for which local input has just been generated
- `nettics[node]` — the highest tic for which each remote node has sent commands

**`TryRunTics`:**

Called each frame. Computes how many tics need to run (based on `I_GetTime()` vs `gametic`). Before running, calls `NetUpdate` to send pending ticcmds and receive those from other nodes. Only advances `gametic` if `nettics[node] > gametic` for all active nodes. Runs as many tics as available (or at least one), calling `G_Ticker` for each.

**`NetUpdate`:**

Sends new ticcmds (from `maketic` forward) to all other nodes. Receives incoming packets and updates `nettics[remotenode]`. Handles retransmit requests (bit `NCMD_RETRANSMIT` in the checksum field) by resending packets. Also calls `D_ProcessEvents` and `G_BuildTiccmd` to generate this frame's local command.

#### Network arbitration (`D_CheckNetGame`)

Before gameplay starts, all nodes exchange setup packets via `D_ArbitrateNetStart`. This establishes `numnodes`, `numplayers`, `consoleplayer` assignment, and initial game parameters (episode, map, skill, deathmatch mode). The `doomcom_t` structure is populated during this phase.

#### Platform network interface (`i_net.c`)

`i_net.c` provides `I_InitNetwork()` (detects and initializes the doomcom interface) and `I_NetCmd()` (triggers the external network driver ISR via a software interrupt). The Linux implementation may use a shared memory block or socket.

#### External network drivers (`ipx/`, `sersrc/`)

The DOS network drivers are separate TSR (Terminate and Stay Resident) programs. They:
1. Allocate a `doomcom_t` structure in shared DOS memory.
2. Hook a software interrupt vector.
3. Launch the DOOM executable with `-net <intnum> <node info>`.
4. When DOOM executes `int <intnum>`, the TSR handles the command:
   - `CMD_SEND`: transmits `doomcom.data` to `doomcom.remotenode`
   - `CMD_GET`: checks for received packets, fills `doomcom.data` and sets `doomcom.remotenode`

The `doomcom_t` structure is identical between the engine and the drivers:
```c
typedef struct {
    long   id;           // DOOMCOM_ID = 0x12345678
    short  intnum;       // software interrupt to invoke
    short  command;      // CMD_SEND or CMD_GET
    short  remotenode;   // target/source node
    short  datalength;   // bytes valid in data[]
    short  numnodes;     // total nodes in game
    short  ticdup;       // tic duplication factor
    short  extratics;    // backup tic count
    short  deathmatch;
    short  savegame;
    short  episode, map, skill;
    short  consoleplayer;
    short  numplayers;
    short  angleoffset;  // for multi-display mode
    short  drone;
    char   data[512];    // raw packet payload
} doomcom_t;
```

**IPX driver (`ipx/`):** Uses Novell NetWare IPX APIs (INT 7A / AES) for LAN play. `IPXSETUP.C` scans for other DOOM instances on the LAN via broadcast, negotiates player order, and launches DOOM.

**Serial driver (`sersrc/`):** Uses direct COM port I/O for two-player modem or null-modem play. `PORT.C` handles UART programming. Supports standard modem AT commands for dial and answer.

---

### 3.16 Fixed-Point Math and Tables (`m_fixed.c/h`, `tables.c/h`)

#### Fixed-point arithmetic

All positions, sizes, momenta, and angles in DOOM use **16.16 fixed-point** numbers:

```c
#define FRACBITS  16
#define FRACUNIT  (1 << FRACBITS)  // = 65536
typedef int fixed_t;
```

One `FRACUNIT` represents 1.0. A player height of 56 units is stored as `56 * FRACUNIT = 3670016`.

- **`FixedMul(a, b)`** — `(long long)a * b >> FRACBITS`. Uses 64-bit intermediate.
- **`FixedDiv(a, b)`** — checks for overflow, returns `(long long)a << FRACBITS / b`. Falls back to `FixedDiv2` which uses the `idivl` assembly instruction on x86.

#### Trigonometric tables

DOOM uses **Binary Angle Measurement (BAM)**: `angle_t` is a 32-bit unsigned integer where `0x100000000` = 360 degrees.

Constants:
```c
#define ANG45   0x20000000
#define ANG90   0x40000000
#define ANG180  0x80000000
#define ANG270  0xc0000000
```

The **fine-angle** system divides the full circle into `FINEANGLES = 8192` subdivisions. An `angle_t` is converted to a fine angle by right-shifting 19 bits (`ANGLETOFINESHIFT = 19`):
```c
fineangle = angle >> ANGLETOFINESHIFT;
```

**Lookup tables** (defined in `tables.c`, declared in `tables.h`):

| Table | Size | Description |
|-------|------|-------------|
| `finesine[5*FINEANGLES/4]` | 10240 entries | Sine table covering 360+90 degrees (the extra 90° allows `finecosine = &finesine[FINEANGLES/4]`) |
| `finetangent[FINEANGLES/2]` | 4096 entries | Tangent table for the first two quadrants |
| `tantoangle[SLOPERANGE+1]` | 2049 entries | Arc-tangent: maps `tan * SLOPERANGE` to `angle_t` |

**`R_PointToAngle(x, y)`** computes the angle from the viewpoint to world coordinates. It reduces to the first octant using absolute values and sign checks, then looks up `tantoangle[SlopeDiv(num, den)]` (where `SlopeDiv` clamps the ratio to `SLOPERANGE = 2048`).

---

## 4. Main Program Flows

### 4.1 Startup Flow

```
main() [i_main.c]
  └─ D_DoomMain() [d_main.c]
       ├─ FindResponseFile()        — expand @response file arguments
       ├─ IdentifyVersion()         — find IWAD, set gamemode, add to wadfiles[]
       ├─ [parse command-line args] — skill, episode, map, -file, -warp, etc.
       ├─ V_Init()                  — allocate screens[0..4] (320x200 byte arrays)
       ├─ M_LoadDefaults()          — parse ~/.doomrc key bindings and settings
       ├─ Z_Init()                  — allocate heap via I_ZoneBase(), init zone
       ├─ W_InitMultipleFiles()     — open WAD files, build lumpinfo[] table
       ├─ M_Init()                  — init switch/platform tables, menu system
       ├─ R_Init()                  — load/compose textures, init sprite tables,
       │                              build viewangle tables and light tables
       ├─ P_Init()                  — init switch texture list
       ├─ I_Init()                  — platform: timer, joystick
       ├─ D_CheckNetGame()          — negotiate with other nodes via doomcom,
       │                              set consoleplayer, numplayers
       ├─ S_Init()                  — allocate sound channels, set volumes
       ├─ HU_Init()                 — load HU font patches
       ├─ ST_Init()                 — load status bar patches
       └─ [start game]
            ├─ G_InitNew()  (autostart or netgame)
            └─ D_StartTitle() (no autostart → demo loop)
       └─ D_DoomLoop()             — enter main loop, never returns
```

### 4.2 Main Game Loop (Per-Tic)

```
D_DoomLoop() [d_main.c]
  while(1):
    I_StartFrame()                  — frame-sync I/O (vsync wait etc.)
    TryRunTics() [d_net.c]
      ├─ [determine how many tics needed based on I_GetTime()]
      ├─ NetUpdate()
      │    ├─ I_StartTic()          — poll input devices → D_PostEvent()
      │    ├─ D_ProcessEvents()     — drain event queue through responder chain
      │    ├─ G_BuildTiccmd()       — build ticcmd from current input state
      │    ├─ [send ticcmds to other nodes via I_NetCmd()]
      │    └─ [receive ticcmds from other nodes, update nettics[]]
      └─ [for each tic where nettics[all] > gametic]:
           G_Ticker() [g_game.c]
             ├─ [process pending gameaction: load level, save, etc.]
             ├─ M_Ticker()          — menu animation
             ├─ [per-state ticker]:
             │   GS_LEVEL:   P_Ticker() → P_PlayerThink() × MAXPLAYERS
             │                           → P_RunThinkers()
             │                           → P_UpdateSpecials()
             │   GS_INTERMISSION: WI_Ticker()
             │   GS_FINALE:  F_Ticker()
             │   GS_DEMOSCREEN: D_PageTicker()
             └─ gametic++
    S_UpdateSounds(listener)        — update positional audio per frame
    D_Display() [d_main.c]         — render (see 4.3)
    I_UpdateSound() / I_SubmitSound() — audio output
```

### 4.3 Rendering Pipeline (Per Frame)

```
D_Display() [d_main.c]
  ├─ [handle view size changes: R_ExecuteSetViewSize()]
  ├─ [save screen for wipe if gamestate changed]
  ├─ [per gamestate drawing]:
  │   GS_LEVEL:
  │     ├─ AM_Drawer()   (if automap active)
  │     ├─ ST_Drawer()   (status bar)
  │     └─ [game view if not automap]:
  │          R_RenderPlayerView(player) [r_main.c]
  │            ├─ R_SetupFrame()        — set viewx/y/z/angle, lighting
  │            ├─ R_ClearClipSegs()     — reset solid seg clipping list
  │            ├─ R_ClearDrawSegs()     — reset drawsegs[] array
  │            ├─ R_ClearPlanes()       — reset visplanes[] array
  │            ├─ R_ClearSprites()      — reset vissprites list
  │            ├─ R_RenderBSPNode()     — [recursive BSP traversal]
  │            │    for each subsector:
  │            │      R_AddSprites()    — project visible sprites → vissprites
  │            │      for each seg:
  │            │        R_AddLine()     — clip to view, R_StoreWallRange()
  │            │          R_StoreWallRange() — compute columns, update solid segs
  │            │                              — create visplanes for floors/ceilings
  │            ├─ R_DrawPlanes()        — draw all visplane floor/ceiling spans
  │            └─ R_DrawMasked()        — draw vissprites back-to-front,
  │                                       then masked wall segments
  │   GS_INTERMISSION: WI_Drawer()
  │   GS_FINALE:       F_Drawer()
  │   GS_DEMOSCREEN:   D_PageDrawer()
  ├─ HU_Drawer()       (messages overlay, in GS_LEVEL)
  ├─ M_Drawer()        (menu always on top)
  ├─ NetUpdate()       (send any pending network data)
  └─ I_FinishUpdate()  (blit to screen / page flip)
```

### 4.4 Player Input Flow

```
[hardware event] → I_StartTic() [i_system.c / i_video.c]
  └─ X11 event → D_PostEvent(ev)   — enqueue in events[] ring buffer

D_ProcessEvents()
  └─ for each event:
       M_Responder(ev)   — menu eats: adjust volumes, navigate, select
       G_Responder(ev)   — game eats:
         ├─ HU_Responder()   — chat input
         ├─ ST_Responder()   — cheat codes
         ├─ AM_Responder()   — automap pan/zoom
         ├─ F_Responder()    — finale skip
         └─ [store in gamekeydown[], mousex/y, joyxmove/y]

G_BuildTiccmd(cmd)
  ├─ read gamekeydown[key_up/down/left/right/fire/use/strafe/speed]
  ├─ read mousebuttons[], mousex, mousey
  ├─ read joybuttons[], joyxmove, joyymove
  ├─ compute forward += mousey, side += mousex (or turn if not strafing)
  ├─ clamp to MAXPLMOVE
  └─ fill cmd: forwardmove, sidemove, angleturn, buttons

[tic execution]
P_PlayerThink(player) [p_user.c]
  ├─ P_MovePlayer(player)
  │    ├─ mo->angle += cmd->angleturn << 16
  │    ├─ forward = cmd->forwardmove * SLOWTURNTICS
  │    ├─ side = cmd->sidemove
  │    ├─ P_Thrust(mo, angle, forward)    — add to momx/momy
  │    └─ P_Thrust(mo, angle-ANG90, side)
  ├─ P_CalcHeight(player)   — compute viewz with bob
  ├─ [fire weapon if BT_ATTACK: P_FireWeapon()]
  ├─ [use line if BT_USE: P_UseLines()]
  └─ [weapon state machine: P_MovePsprites()]

P_MobjThinker(mo) [p_mobj.c]
  ├─ [apply gravity if !MF_NOGRAVITY]
  ├─ P_XYMovement(mo)     — apply momx/momy, P_TryMove(), friction
  └─ P_ZMovement(mo)      — apply momz, floor/ceiling collision
```

### 4.5 Monster AI Tick

```
P_Ticker() [p_tick.c]
  └─ P_RunThinkers()
       └─ for each thinker in thinkercap list:
            P_MobjThinker(mo) [p_mobj.c]
              ├─ [apply momentum / gravity]
              └─ [decrement tics; if tics == 0: P_SetMobjState(mo, state->nextstate)]
                   └─ state->action()  — e.g.:

A_Look(mo) [p_enemy.c]
  ├─ check sector->soundtarget (heard player via sound propagation)
  └─ P_LookForPlayers(mo)
       └─ for nearby players: P_CheckSight(mo, player->mo)
            → set mo->target, transition to SEE state

A_Chase(mo) [p_enemy.c]
  ├─ [decrement reactiontime]
  ├─ [check target still alive and reachable]
  ├─ P_Move(mo)         — take one step in mo->movedir
  │    └─ P_TryMove()   — attempt the move
  ├─ [if move fails: P_NewChaseDir()]  — select new direction
  └─ [if in attack range: transition to ATTACK state → A_*Attack()]

A_*Attack(mo) [p_enemy.c]
  ├─ A_FaceTarget(mo)   — rotate toward target
  ├─ [hitscan: P_LineAttack() / P_AimLineAttack()]
  └─ [projectile: P_SpawnMissile()]
```

### 4.6 Map Loading Flow

```
G_Ticker() — processes ga_loadlevel action
  └─ G_DoLoadLevel() [g_game.c]
       ├─ [set sky texture based on episode/map]
       ├─ gamestate = GS_LEVEL
       ├─ [reset player states]
       └─ P_SetupLevel(episode, map, 0, skill) [p_setup.c]
            ├─ Z_FreeTags(PU_LEVEL, PU_PURGELEVEL)  — free previous level
            ├─ P_InitThinkers()      — reset thinker list
            ├─ [compute lump base index for this map's label]
            ├─ P_LoadVertexes(lumpnum + ML_VERTEXES)
            ├─ P_LoadSectors(lumpnum + ML_SECTORS)
            ├─ P_LoadSideDefs(lumpnum + ML_SIDEDEFS)
            ├─ P_LoadLineDefs(lumpnum + ML_LINEDEFS)
            ├─ P_LoadSegs(lumpnum + ML_SEGS)
            ├─ P_LoadSubsectors(lumpnum + ML_SSECTORS)
            ├─ P_LoadNodes(lumpnum + ML_NODES)
            ├─ P_LoadReject(lumpnum + ML_REJECT)
            ├─ P_GroupLines()        — cross-reference lines↔sectors
            ├─ P_LoadBlockMap(lumpnum + ML_BLOCKMAP)
            ├─ P_SpawnSpecials()     — create thinkers for active sector specials
            ├─ P_LoadThings(lumpnum + ML_THINGS)  — spawn all mobj_t
            │    └─ P_SpawnMapThing() for each mapthing_t
            ├─ P_SpawnPlayer() for each playerstart
            └─ [if precache: cache all textures/sprites/flats via R_PrecacheLevel]
```

### 4.7 Save/Load Flow

```
[save triggered by menu or BTS_SAVEGAME button]
G_DoSaveGame() [g_game.c]
  ├─ savebuffer = Z_Malloc(SAVEGAMESIZE, PU_STATIC, 0)
  ├─ P_ArchivePlayers()    — serialize all player_t structs
  ├─ P_ArchiveWorld()      — serialize sector heights, line special data
  ├─ P_ArchiveThinkers()   — walk thinkercap list, serialize each mobj_t
  ├─ P_ArchiveSpecials()   — serialize moving floors, ceilings, doors, etc.
  ├─ [write save_p - savebuffer bytes to doomsavN.dsg]
  └─ Z_Free(savebuffer)

G_DoLoadGame() [g_game.c]
  ├─ [read doomsavN.dsg into savebuffer]
  ├─ G_InitNew(skill, episode, map)   — set up game parameters
  ├─ G_DoLoadLevel()                  — load the level geometry fresh
  ├─ P_UnArchivePlayers()             — restore player_t structs
  ├─ P_UnArchiveWorld()               — restore sector/line state
  ├─ P_UnArchiveThinkers()            — recreate all mobj_t, fix up pointers
  └─ P_UnArchiveSpecials()            — recreate sector thinkers
```

### 4.8 Networking Flow

```
D_CheckNetGame() [d_net.c / i_net.c]
  ├─ I_InitNetwork()      — detect doomcom structure from driver
  ├─ D_ArbitrateNetStart()
  │    └─ exchange NCMD_SETUP packets with all nodes until all ready
  │       populate: consoleplayer, numplayers, episode/map/skill
  └─ [set ticdup, maxsend based on network speed]

[each frame]
TryRunTics() [d_net.c]
  ├─ NetUpdate()
  │    ├─ I_StartTic() → D_PostEvent() — collect input
  │    ├─ D_ProcessEvents() + G_BuildTiccmd() — build local ticcmd
  │    ├─ [store in localcmds[maketic % BACKUPTICS]; maketic++]
  │    ├─ [build doomdata_t packet: starttic, player, numtics, cmds[]]
  │    ├─ [for each active remote node: doomcom->command = CMD_SEND;
  │    │    doomcom->remotenode = node; I_NetCmd() → driver ISR]
  │    └─ [while CMD_GET returns packets:
  │         update nettics[remotenode] = max(nettics, packet.starttic + numtics)]
  └─ [run tics while all nettics[node] > gametic]
       G_Ticker() for each tic
```

---

## 5. Key Data Structures

### `mobj_t` (Map Object / Thing) — `p_mobj.h`

The universal actor. Head is a `thinker_t` (must be first member for thinker list interoperability). Holds 3D position in fixed-point, sector/blockmap links, momentum, sprite/frame for rendering, health, state machine pointer, AI fields (`target`, `movedir`, `threshold`), and a back-pointer to `player_t` for player-controlled objects.

**Notable flags** (`mobjflag_t`):
- `MF_SOLID` — blocks movement
- `MF_SHOOTABLE` — can be damaged
- `MF_NOBLOCKMAP` — not in blockmap (inert/decorative)
- `MF_NOSECTOR` — not in sector list (invisible but touchable)
- `MF_MISSILE` — projectile behavior
- `MF_FLOAT` — active floater (cacodemon, pain elemental)
- `MF_SHADOW` — fuzzy/spectre rendering
- `MF_COUNTKILL` / `MF_COUNTITEM` — counts toward kill/item totals

### `sector_t`, `line_t`, `side_t`, `seg_t`, `subsector_t`, `node_t` — `r_defs.h`

Map geometry structures:

- **`sector_t`** — defines a convex region with floor/ceiling heights, flat texture indices, light level, special type, and tag. Contains `thinglist` (linked list of `mobj_t` in the sector), `soundtarget` (last heard thing), and `specialdata` (active thinker pointer).
- **`line_t`** — a wall edge between two vertices. Has flags (blocking, two-sided, peg flags), special trigger code, tag, two sidenum indices, front/back sector pointers, bounding box, and slope type.
- **`side_t`** — the surface of one side of a linedef. Stores texture indices (top, mid, bottom) and texture offsets (`textureoffset`, `rowoffset`).
- **`seg_t`** — a BSP-split fragment of a linedef. Contains vertex pointers, angle, side/linedef references, front/back sector references, and texture column offset.
- **`subsector_t`** — a BSP leaf: a convex polygon defined by a contiguous range of segs. References its parent `sector_t`.
- **`node_t`** — a BSP tree internal node. Contains the partition line (`x, y, dx, dy`), bounding boxes for each child subtree, and child indices (with `NF_SUBSECTOR` bit indicating a leaf).

### `player_t` — `d_player.h`

Player-specific state extending `mobj_t`. Contains:
- `mo` — the associated mobj
- `playerstate` — `PST_LIVE`, `PST_DEAD`, `PST_REBORN`
- `cmd` — the current `ticcmd_t`
- `viewz`, `viewheight`, `deltaviewheight`, `bob` — view height computation
- Inventory: `health`, `armorpoints`, `armortype`, `powers[NUMPOWERS]`, `cards[NUMCARDS]`, `weaponowned[NUMWEAPONS]`, `ammo[NUMAMMO]`
- `readyweapon`, `pendingweapon`
- `frags[MAXPLAYERS]` — kill counts in deathmatch
- `cheats` — `CF_NOCLIP`, `CF_GODMODE`, `CF_NOMOMENTUM`
- `psprites[NUMPSPRITES]` — weapon overlay sprites (gun bob, muzzle flash)
- `damagecount`, `bonuscount` — screen flash timers
- `extralight` — extra sector light from gun flash

### `state_t` / `mobjinfo_t` — `info.h` / `info.c`

The thing type database:

**`state_t`** — one animation frame in a state machine:
- `sprite` (`spritenum_t`), `frame` (frame index, may have `FF_FULLBRIGHT`)
- `tics` — duration (-1 = infinite)
- `action` (`actionf_t`) — function called when this state is entered
- `nextstate` (`statenum_t`) — next state index

**`mobjinfo_t`** — the static description of a thing type (`mobjtype_t` enum, e.g., `MT_PLAYER`, `MT_TROOPER`):
- Spawn/see/pain/death/raise/missile/melee state indices
- `doomednum` — editor thing number
- `spawnhealth`, `reactiontime`, `painchance`, `speed`, `radius`, `height`, `mass`
- `damage` (for contact damage)
- `flags` (`mobjflag_t` combination)
- `activesound`, `attacksound`, `painsound`, `deathsound`, `seesound`

### `memblock_t` / `memzone_t` — `z_zone.c/h`

The zone allocator structures. `memzone_t` holds the sentinel `blocklist` and the `rover` pointer. `memblock_t` is the header prepended to every allocation: `size`, `user` (pointer-to-pointer for auto-null on eviction), `tag`, `id` (= `ZONEID = 0x1d4a11` for validation), and doubly-linked `next`/`prev`.

### `wadinfo_t` / `lumpinfo_t` — `w_wad.h`

`wadinfo_t` is the 12-byte WAD header. `lumpinfo_t` is the runtime lump descriptor: name (8 chars), file handle, byte position, size. The `lumpcache[]` parallel array holds cached `void*` pointers (or NULL if not loaded).

### `vissprite_t` — `r_defs.h`

Projected sprite ready for rendering. Contains screen X extents (`x1`, `x2`), world position, scale, texture start fraction and step (`startfrac`, `xiscale`), texture middle height (`texturemid`), patch lump number, colormap pointer (for lighting and translation), and `mobjflags` (for shadow/fuzzy detection). Stored in a doubly-linked list sorted by scale.

### `drawseg_t` — `r_defs.h`

Records a rendered wall segment. Stores the source `seg_t*`, screen column range (`x1`, `x2`), scale at both ends and step, silhouette type (for sprite clipping), silhouette heights, and pointers into the `openings[]` array for sprite top/bottom clip arrays and masked texture column indices.

### `visplane_t` — `r_defs.h`

A horizontal surface to be drawn. Stores `height`, `picnum` (flat texture), `lightlevel`, column extent `minx`/`maxx`, and per-column top/bottom pixel row arrays (`top[SCREENWIDTH]`, `bottom[SCREENWIDTH]`). Up to `MAXVISPLANES = 128` per frame.

### `ticcmd_t` — `d_ticcmd.h`

The network-transmitted input command for one game tic:
```c
typedef struct {
    char  forwardmove;   // forward/backward speed (signed, *2048 = fixed_t)
    char  sidemove;      // strafe speed (signed)
    short angleturn;     // turn amount (<<16 = angle_t delta)
    short consistancy;   // XOR checksum of game state for net validation
    byte  chatchar;      // pending chat character (0 = none)
    byte  buttons;       // BT_ATTACK, BT_USE, BT_SPECIAL | BTS_* flags
} ticcmd_t;
```

### `doomcom_t` — `d_net.h`

The shared memory block between DOOM and an external network driver. Identified by `id = DOOMCOM_ID = 0x12345678`. The driver exposes a software interrupt number (`intnum`); DOOM signals the driver by executing `int intnum`. Fields: `command` (send/get), `remotenode`, `datalength`, game parameters, and `data` (the raw `doomdata_t` packet payload).

---

## 6. File Index

### `linuxdoom-1.10/`

| File | Description |
|------|-------------|
| `i_main.c` | Program entry point; stores argc/argv and calls `D_DoomMain` |
| `i_system.c` | Platform system functions: `I_GetTime`, `I_Error`, `I_ZoneBase`, `I_Quit` |
| `i_system.h` | Platform system interface declarations |
| `i_video.c` | X11 video output: `I_InitGraphics`, `I_FinishUpdate`, `I_StartTic` (X event polling) |
| `i_video.h` | Video interface declarations |
| `i_sound.c` | Linux OSS sound interface; SNDSERV pipe communication or direct DSP mixing |
| `i_sound.h` | Sound platform interface declarations |
| `i_net.c` | Network platform layer: detect doomcom, `I_NetCmd` interrupt call |
| `i_net.h` | Network platform interface declarations |
| `d_main.c` | `D_DoomMain` startup, `D_DoomLoop` main loop, `D_Display`, event ring buffer |
| `d_main.h` | Declarations for `D_DoomMain`, `D_DoomLoop`, `D_PostEvent` |
| `d_net.c` | Networking: `TryRunTics`, `NetUpdate`, `D_CheckNetGame`, `D_ArbitrateNetStart` |
| `d_net.h` | Network data structures: `doomcom_t`, `doomdata_t`, `ticcmd_t` protocol; `TryRunTics` declaration |
| `d_event.h` | `event_t` structure and `evtype_t` enum (keyboard, mouse, joystick) |
| `d_items.c` | Weapon and ammo pickup definitions |
| `d_items.h` | `weaponinfo_t` structure and weapon info array declaration |
| `d_player.h` | `player_t`, `playerstate_t`, `wbstartstruct_t`, `wbplayerstruct_t` |
| `d_think.h` | `thinker_t`, `actionf_t` function pointer union; the base of all active objects |
| `d_textur.h` | Texture type identifiers (unused/minimal) |
| `d_ticcmd.h` | `ticcmd_t` structure |
| `doomdef.h` | Core constants: `VERSION`, `SCREENWIDTH/HEIGHT`, `TICRATE`, `MAXPLAYERS`; `gamestate_t`, `skill_t`, weapon/ammo/powerup enums; key codes |
| `doomdef.c` | (Minimal; versioning) |
| `doomstat.c` | Definitions of global game state variables (gamemode, gameskill, players array, etc.) |
| `doomstat.h` | Declarations for all global game state variables |
| `doomdata.h` | WAD map binary format structs: `mapvertex_t`, `mapsidedef_t`, `maplinedef_t`, `mapsector_t`, `mapseg_t`, `mapsubsector_t`, `mapnode_t`, `mapthing_t`; `ML_*` lump offset constants |
| `doomtype.h` | Basic types: `boolean`, `byte`, `ushort`, `uint` |
| `dstrings.c` | String definitions for all in-game messages |
| `dstrings.h` | Declarations for message strings |
| `d_englsh.h` | English-language string constants |
| `d_french.h` | French-language string constants |
| `g_game.c` | Game state machine: `G_Ticker`, `G_BuildTiccmd`, `G_DoLoadLevel`, `G_DoSaveGame`, `G_DoLoadGame`, `G_PlayDemo`, `G_RecordDemo`, `G_Responder` |
| `g_game.h` | `gameaction_t` enum; declarations for game control functions |
| `r_main.c` | Renderer entry: `R_RenderPlayerView`, `R_Init`, view setup, BSP traversal launch, angle/projection tables |
| `r_main.h` | Renderer globals: viewx/y/z/angle, projection, light tables, `R_PointToAngle` |
| `r_bsp.c` | BSP tree traversal: `R_RenderBSPNode`, `R_Subsector`, `R_AddLine`, `R_CheckBBox`; solid seg clipping |
| `r_bsp.h` | BSP renderer declarations |
| `r_segs.c` | Wall rendering: `R_StoreWallRange`, scale computation, column generation, visplane creation |
| `r_segs.h` | Wall renderer declarations |
| `r_plane.c` | Floor/ceiling rendering: visplane management, `R_DrawPlanes`, span drawing, sky handling |
| `r_plane.h` | Plane renderer declarations |
| `r_things.c` | Sprite rendering: `R_AddSprites`, `R_ProjectSprite`, `R_DrawMasked`, `R_DrawVisSprite` |
| `r_things.h` | Sprite renderer declarations |
| `r_draw.c` | Pixel-level drawing: `R_DrawColumn`, `R_DrawFuzzColumn`, `R_DrawTranslatedColumn`, `R_DrawSpan` |
| `r_draw.h` | Drawing function declarations and column/span parameters |
| `r_data.c` | Texture data preparation: `R_InitTextures`, `R_InitFlats`, `R_InitSprites`, `R_GetColumn`, `R_PrecacheLevel` |
| `r_data.h` | Texture/flat/sprite data declarations |
| `r_sky.c` | Sky texture tracking (`skytexture`, `skyflatnum`) |
| `r_sky.h` | Sky declarations |
| `r_defs.h` | Shared renderer/play data structures: `vertex_t`, `sector_t`, `side_t`, `line_t`, `seg_t`, `subsector_t`, `node_t`, `drawseg_t`, `vissprite_t`, `visplane_t`, `patch_t`, `spriteframe_t` |
| `r_local.h` | Internal renderer constants and declarations |
| `r_state.h` | Extern declarations for renderer global state arrays (textures, sprites, etc.) |
| `p_setup.c` | Level loading: `P_SetupLevel`, `P_LoadVertexes/Sectors/LineDefs/SideDefs/Segs/Nodes/Blockmap/Reject/Things` |
| `p_setup.h` | `P_SetupLevel` declaration |
| `p_mobj.c` | Mobj lifecycle: `P_SpawnMobj`, `P_RemoveMobj`, `P_MobjThinker`, `P_SetMobjState`, `P_SpawnMapThing`, `P_XYMovement`, `P_ZMovement` |
| `p_mobj.h` | `mobj_t` structure, `mobjflag_t` enum |
| `p_tick.c` | Thinker list management: `P_RunThinkers`, `P_AddThinker`, `P_RemoveThinker`, `P_Ticker` |
| `p_tick.h` | Thinker function declarations |
| `p_map.c` | Movement and collision: `P_TryMove`, `P_CheckPosition`, `P_SlideMove`, `P_LineAttack`, `P_AimLineAttack`, `P_UseLines`, `P_RadiusAttack` |
| `p_maputl.c` | Map traversal utilities: `P_PathTraverse`, `P_BlockThingsIterator`, `P_BlockLinesIterator`, `P_PointOnLineSide`, `P_InterceptVector` |
| `p_user.c` | Player physics: `P_MovePlayer`, `P_CalcHeight`, `P_DeathThink`, `P_PlayerThink` |
| `p_enemy.c` | Monster AI: `A_Look`, `A_Chase`, `A_FaceTarget`, `P_Move`, `P_NewChaseDir`, all `A_*Attack` action functions |
| `p_inter.c` | Interaction callbacks: `P_TouchSpecialThing` (pickups), `P_DamageMobj`, `P_KillMobj` |
| `p_inter.h` | Interaction function declarations |
| `p_spec.c` | Sector specials: `P_CrossSpecialLine`, `P_UseSpecialLine`, `P_ShootSpecialLine`, `P_UpdateSpecials`, `P_SpawnSpecials`, `P_RecursiveSound` |
| `p_spec.h` | Special type enums and function declarations |
| `p_ceilng.c` | Ceiling movement thinker: `T_MoveCeiling`, `EV_CrushCeiling`, `EV_CeilingCrushStop` |
| `p_doors.c` | Door thinker: `T_VerticalDoor`, `EV_VerticalDoor`, `EV_DoLockedDoor` |
| `p_floor.c` | Floor movement: `T_MoveFloor`, `T_MovePlane`, `EV_BuildStairs`, `EV_DoFloor` |
| `p_lights.c` | Lighting effects: `T_FireFlicker`, `T_LightFlash`, `T_StrobeFlash`, `T_Glow`, spawn functions |
| `p_plats.c` | Platform thinker: `T_PlatRaise`, `EV_DoPlat`, `EV_StopPlat` |
| `p_switch.c` | Switch texture cycling: `P_InitSwitchList`, `P_ChangeSwitchTexture` |
| `p_telept.c` | Teleporter: `EV_Teleport` |
| `p_sight.c` | Line-of-sight: `P_CheckSight`, `P_CrossSubsectors`, `P_CrossBSPNode` |
| `p_pspr.c` | Player sprite (weapon) state machine: `P_SetPsprite`, `P_MovePsprites`, `P_FireWeapon`, `A_WeaponReady`, `A_ReFire`, `A_Lower`, `A_Raise`, weapon fire action functions |
| `p_pspr.h` | `pspdef_t` (player sprite state), NUMPSPRITES, declarations |
| `p_saveg.c` | Save/load serialization: `P_ArchivePlayers/World/Thinkers/Specials`, `P_UnArchive*` |
| `p_saveg.h` | Save/load declarations |
| `p_local.h` | Internal play module constants and declarations (MAXSTEP, USERANGE, MELEERANGE, MISSILERANGE, friction, etc.) |
| `m_argv.c` | `M_CheckParm` command-line argument lookup |
| `m_argv.h` | `myargc`, `myargv`, `M_CheckParm` declarations |
| `m_bbox.c` | Axis-aligned bounding box utilities |
| `m_bbox.h` | Bounding box macros (BOXLEFT, BOXRIGHT, BOXTOP, BOXBOTTOM) |
| `m_cheat.c` | Cheat code recognition via partial string matching state machine |
| `m_cheat.h` | `cheatseq_t` structure and `M_CheatResponder` declaration |
| `m_fixed.c` | `FixedMul`, `FixedDiv`, `FixedDiv2` implementations (with x86 inline asm) |
| `m_fixed.h` | `fixed_t` typedef, `FRACBITS`, `FRACUNIT`, function declarations |
| `m_menu.c` | Complete menu system: all menu definitions, `M_Responder`, `M_Ticker`, `M_Drawer`, `M_StartControlPanel` |
| `m_menu.h` | `M_Responder`, `M_Ticker`, `M_Drawer` declarations; menu flag |
| `m_misc.c` | Config file I/O (`M_LoadDefaults`, `M_SaveDefaults`), screenshot (`M_ScreenShot`), `M_WriteFile`/`M_ReadFile` |
| `m_misc.h` | Misc function declarations |
| `m_random.c` | Deterministic pseudo-random number table and `P_Random()` / `M_Random()` functions |
| `m_random.h` | `P_Random`, `M_Random` declarations; `rndindex` |
| `m_swap.c` | Byte-order swap functions `SHORT` / `LONG` (no-ops on little-endian) |
| `m_swap.h` | Endianness macros |
| `am_map.c` | Automap: coordinate transforms, line drawing, player arrow, `AM_Responder`, `AM_Ticker`, `AM_Drawer` |
| `am_map.h` | Automap declarations, `automapactive` flag |
| `st_stuff.c` | Status bar: all widget definitions, `ST_Init`, `ST_Ticker`, `ST_Drawer`, `ST_Responder`, face animation |
| `st_stuff.h` | Status bar declarations |
| `st_lib.c` | Status bar widget library: numbers, percentages, binary icons, multi-icons |
| `st_lib.h` | Widget structure types and function declarations |
| `hu_stuff.c` | HUD: message display, chat system, map name overlay, `HU_Init`, `HU_Ticker`, `HU_Drawer`, `HU_Responder`, `HU_Erase` |
| `hu_stuff.h` | HUD declarations |
| `hu_lib.c` | HUD widget primitives: `hu_textline_t`, `hu_stext_t` (scrolling message), `hu_itext_t` (input text) |
| `hu_lib.h` | HUD widget structure types |
| `wi_stuff.c` | Intermission screen: statistics display, episode map animation, `WI_Start`, `WI_Ticker`, `WI_Drawer` |
| `wi_stuff.h` | Intermission declarations |
| `f_finale.c` | Episode finale: text scroll, cast of characters, `F_StartFinale`, `F_Ticker`, `F_Drawer`, `F_Responder` |
| `f_finale.h` | Finale declarations |
| `f_wipe.c` | Screen wipe transitions: `wipe_StartScreen`, `wipe_EndScreen`, `wipe_ScreenWipe` (melt effect) |
| `f_wipe.h` | Wipe function declarations and wipe type enum |
| `v_video.c` | Video utilities: `V_Init` (allocate screens[]), `V_DrawPatch`, `V_DrawPatchDirect`, `V_DrawBlock`, `V_MarkRect`, `V_CopyRect` |
| `v_video.h` | Video utility declarations; `screens[]` extern |
| `w_wad.c` | WAD I/O: `W_InitMultipleFiles`, `W_CheckNumForName`, `W_GetNumForName`, `W_CacheLumpNum`, `W_ReadLump` |
| `w_wad.h` | WAD types (`wadinfo_t`, `filelump_t`, `lumpinfo_t`) and function declarations |
| `z_zone.c` | Zone allocator: `Z_Init`, `Z_Malloc`, `Z_Free`, `Z_FreeTags`, `Z_ChangeTag2`, `Z_CheckHeap` |
| `z_zone.h` | Zone public API, purge tag constants, `memblock_t` structure, `Z_ChangeTag` macro |
| `s_sound.c` | Sound channel management: `S_Init`, `S_StartSound`, `S_StopSound`, `S_UpdateSounds`, `S_StartMusic`, `S_ChangeMusic` |
| `s_sound.h` | Sound management function declarations |
| `info.c` | All `state_t` and `mobjinfo_t` data: complete thing type database for all monsters, items, and decorations |
| `info.h` | `spritenum_t`, `statenum_t`, `mobjtype_t` enums; `state_t`, `mobjinfo_t` structures; extern table declarations |
| `sounds.c` | `sfxinfo_t` array (`S_sfx[]`) and `musicinfo_t` array (`S_music[]`) definitions |
| `sounds.h` | `sfxenum_t` and `musicenum_t` enums; `sfxinfo_t`, `musicinfo_t` structures; array declarations |
| `tables.c` | `finesine[]`, `finetangent[]`, `tantoangle[]` lookup table data |
| `tables.h` | Table declarations, `FINEANGLES`, `angle_t` typedef, BAM angle constants, `SlopeDiv` |

### `sndserv/`

| File | Description |
|------|-------------|
| `soundsrv.c` | Main sound server: IPC command loop, software audio mixer, voxware output |
| `soundsrv.h` | Sample rate constants (`SPEED = 11025`), buffer size, function declarations |
| `linux.c` | Linux-specific audio device management and voxware ioctl calls |
| `sounds.c` | Sound effect and music info tables (copy of linuxdoom version) |
| `sounds.h` | Sound type declarations (copy of linuxdoom version) |
| `soundst.h` | Sound state structure for the server's channel tracking |
| `wadread.c` | Minimal WAD file reader for the sound server to extract sound lumps |
| `wadread.h` | WAD reader declarations |
| `Makefile` | Build system for the standalone sound server binary |
| `README.sndserv` | Documentation for the sound server |

### `ipx/`

| File | Description |
|------|-------------|
| `IPXSETUP.C` | IPX network setup program: LAN discovery, player negotiation, DOOM launch |
| `IPXNET.C` | IPX packet transmission and reception via Novell NetWare INT 7A |
| `IPXNET.H` | IPX network structure definitions |
| `DOOMNET.C` | Shared TSR infrastructure: interrupt hooking, `doomcom_t` setup, `LaunchDOOM`, `NetISR` |
| `DOOMNET.H` | `doomcom_t` structure definition (DOS version); shared constants |
| `IPXSTR.H` | English user-interface strings for IPX setup |
| `IPX_FRCH.H` | French user-interface strings for IPX setup |
| `README` | IPX driver build and usage documentation |

### `sersrc/`

| File | Description |
|------|-------------|
| `SERSETUP.C` | Serial/modem setup: COM port configuration, modem dial/answer, DOOM launch |
| `SERSETUP.H` | Serial setup declarations |
| `PORT.C` | Low-level UART COM port I/O: send/receive bytes, set baud rate |
| `DOOMNET.C` | Shared TSR infrastructure (same role as ipx/DOOMNET.C) |
| `DOOMNET.H` | `doomcom_t` structure (serial version) |
| `SERSTR.H` | English user-interface strings |
| `SER_FRCH.H` | French user-interface strings |
| `README.TXT` | Serial driver documentation |
