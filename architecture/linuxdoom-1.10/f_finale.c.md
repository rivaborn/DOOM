# File Overview

**Source file:** `linuxdoom-1.10/f_finale.c`

`f_finale.c` implements the end-game sequences that play after the player completes the final map of an episode or the entire game. It is responsible for three distinct stages of the finale animation, controlled by the global `finalestage` variable:

- **Stage 0 (text crawl):** Scrolls episode-completion text across a tiled flat background, one character at a time at `TEXTSPEED` (3 tics per character).
- **Stage 1 (art screen):** Displays a full-screen artwork image; for episode 3 of DOOM 1 this is the animated "bunny scroll" (PFUB1/PFUB2) with an animated "END" logo sequence.
- **Stage 2 (cast parade):** DOOM II's end-game "cast of characters" sequence where every monster and the player walk on-screen, are named, and can be killed by the player pressing a key.

The module integrates tightly with the game state machine: it is entered via `F_StartFinale()`, ticked each game tic via `F_Ticker()`, drawn via `F_Drawer()`, and receives input events via `F_Responder()`.

---

## Global Variables

| Type | Name | Purpose |
|------|------|---------|
| `int` | `finalestage` | Current stage of the finale animation: 0 = text crawl, 1 = art screen, 2 = cast parade |
| `int` | `finalecount` | Tic counter since the finale began; drives timing for text reveal, scroll position, and stage transitions |
| `char*` | `finaletext` | Pointer to the text string currently being scrolled onto the screen |
| `char*` | `finaleflat` | Name of the WAD flat lump used to tile the background during the text stage |
| `char*` | `e1text` | Text for DOOM 1 episode 1 completion (initialized from `E1TEXT` in `dstrings.h`) |
| `char*` | `e2text` | Text for DOOM 1 episode 2 completion (`E2TEXT`) |
| `char*` | `e3text` | Text for DOOM 1 episode 3 completion (`E3TEXT`) |
| `char*` | `e4text` | Text for DOOM 1 episode 4 (Ultimate DOOM) completion (`E4TEXT`) |
| `char*` | `c1text` … `c6text` | Text strings for DOOM II inter-boss level completions (`C1TEXT` … `C6TEXT`) |
| `char*` | `p1text` … `p6text` | Text strings for Plutonia pack level completions (`P1TEXT` … `P6TEXT`) |
| `char*` | `t1text` … `t6text` | Text strings for TNT pack level completions (`T1TEXT` … `T6TEXT`) |
| `int` | `castnum` | Index into `castorder[]` identifying the monster currently on display in the cast parade |
| `int` | `casttics` | Remaining tics before advancing to the next animation frame for the current cast member |
| `state_t*` | `caststate` | Pointer into the global `states[]` array; the current animation state of the cast member |
| `boolean` | `castdeath` | True when the current cast member is playing its death animation (triggered by player key press) |
| `int` | `castframes` | Frame counter within the current animation cycle; used to trigger attack animations at frame 12 and reset at frame 24 |
| `int` | `castonmelee` | Toggle flag (XOR'd each attack cycle) selecting between melee and missile attack states |
| `boolean` | `castattacking` | True while the cast member is in its attack animation |

### Compile-time constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `TEXTSPEED` | 3 | Tics between revealing successive characters of `finaletext` |
| `TEXTWAIT` | 250 | Additional tics to wait after all text has been revealed before transitioning to stage 1 |

---

## Data Structures

### `castinfo_t`

```c
typedef struct {
    char       *name;    // Display name shown at the bottom of the screen
    mobjtype_t  type;    // mobj type used to look up states, sounds, and sprites
} castinfo_t;
```

Used to define each entry in the cast parade. The array is terminated by an entry with `name == NULL`.

### `castorder[]` (static array of `castinfo_t`)

The full ordered cast list used in the DOOM II ending:

```
CC_ZOMBIE  / MT_POSSESSED
CC_SHOTGUN / MT_SHOTGUY
CC_HEAVY   / MT_CHAINGUY
CC_IMP     / MT_TROOP
CC_DEMON   / MT_SERGEANT
CC_LOST    / MT_SKULL
CC_CACO    / MT_HEAD
CC_HELL    / MT_KNIGHT
CC_BARON   / MT_BRUISER
CC_ARACH   / MT_BABY
CC_PAIN    / MT_PAIN
CC_REVEN   / MT_UNDEAD
CC_MANCU   / MT_FATSO
CC_ARCH    / MT_VILE
CC_SPIDER  / MT_SPIDER
CC_CYBER   / MT_CYBORG
CC_HERO    / MT_PLAYER
{ NULL, 0 }   (sentinel)
```

---

## Functions

### `F_StartFinale`

```c
void F_StartFinale(void)
```

**Purpose:** Initializes the finale sequence. Called by the game state machine when `gameaction == ga_victory`.

**Key logic:**
1. Sets `gameaction = ga_nothing` and `gamestate = GS_FINALE`; disables the 3D view and automap.
2. Selects `finaleflat` and `finaletext` based on `gamemode` and `gameepisode`/`gamemap`:
   - **shareware / registered / retail:** Uses episode-specific flats (`FLOOR4_8`, `SFLR6_1`, `MFLR8_4`, `MFLR8_3`) and plays `mus_victor`.
   - **commercial (DOOM II):** Uses map-specific flats and plays `mus_read_m`; only triggered at maps 6, 11, 20, 30, 15, 31.
   - **default (indeterminate):** Falls back to `F_SKY1` flat and `c1text`.
3. Resets `finalestage = 0` and `finalecount = 0`.

**Parameters:** none
**Returns:** void

---

### `F_Responder`

```c
boolean F_Responder(event_t *event)
```

**Purpose:** Routes input events to the cast parade handler when in stage 2; ignores events in stages 0 and 1.

**Parameters:**
- `event` — pointer to the input event to handle

**Returns:** `true` if the event was consumed, `false` otherwise.

**Key logic:** Delegates entirely to `F_CastResponder` when `finalestage == 2`.

---

### `F_Ticker`

```c
void F_Ticker(void)
```

**Purpose:** Advances the finale animation each game tic. Called from `G_Ticker` when `gamestate == GS_FINALE`.

**Key logic:**
1. **DOOM II skip check:** If `gamemode == commercial` and `finalecount > 50`, polls all player button states. If any player pressed a button, either starts the cast parade (map 30) or sets `gameaction = ga_worlddone` to advance to the next level.
2. Increments `finalecount`.
3. If `finalestage == 2`, delegates to `F_CastTicker` and returns.
4. For non-commercial mode only: when text has fully scrolled (`finalecount > strlen(finaletext) * TEXTSPEED + TEXTWAIT`) transitions to stage 1:
   - Resets `finalecount = 0` and sets `finalestage = 1`.
   - Sets `wipegamestate = -1` to force a screen wipe.
   - If `gameepisode == 3`, starts the bunny music (`mus_bunny`).

**Returns:** void

---

### `F_TextWrite`

```c
void F_TextWrite(void)
```

**Purpose:** Draws the tiled background flat and renders the text crawl for stage 0.

**External dependency:** Uses `hu_font[]` from `hu_stuff.h` for proportional-width font rendering.

**Key logic:**
1. Tiles the 64×64 flat named by `finaleflat` across the entire screen using `memcpy` into `screens[0]`, handling partial-width right edge.
2. Calls `V_MarkRect` to mark the entire screen dirty.
3. Computes `count = (finalecount - 10) / TEXTSPEED`; this is the number of characters to reveal.
4. Iterates over `finaletext`, character by character up to `count`:
   - Newlines advance `cy` by 11 pixels and reset `cx` to 10.
   - Non-printable characters advance `cx` by 4.
   - Printable characters are upper-cased, offset by `HU_FONTSTART`, and drawn via `V_DrawPatch`.
   - Drawing stops if the glyph would exceed `SCREENWIDTH`.

**Returns:** void

---

### `F_StartCast`

```c
void F_StartCast(void)
```

**Purpose:** Initiates the DOOM II cast parade (stage 2). Called from `F_Ticker` when the player skips the text on map 30.

**Key logic:**
- Forces a screen wipe (`wipegamestate = -1`).
- Initializes `castnum = 0`, sets `caststate` to the first cast member's `seestate`, sets `casttics`, `castframes`, `castonmelee`, `castattacking`, and `castdeath` to their defaults.
- Changes music to `mus_evil`.
- Sets `finalestage = 2`.

**Returns:** void

---

### `F_CastTicker`

```c
void F_CastTicker(void)
```

**Purpose:** Advances the cast parade animation each tic. Called by `F_Ticker` when `finalestage == 2`.

**Key logic:**
1. Decrements `casttics`; returns immediately if time remains in the current frame.
2. **State transition logic:**
   - If the current state has `tics == -1` or `nextstate == S_NULL` (end of death animation), advances to the next cast member: increments `castnum` (wraps to 0 at end of array), plays the new monster's see-sound, resets to `seestate`, resets `castframes`.
   - Otherwise advances to `caststate->nextstate` and increments `castframes`. A large `switch` on the new state index triggers the appropriate attack sound effect.
3. **Attack trigger:** At `castframes == 12`, sets `castattacking = true` and switches the state to either `meleestate` or `missilestate` (alternating via `castonmelee ^= 1`). If the selected state is `S_NULL`, falls back to the other attack type.
4. **Attack reset:** If `castattacking` and either `castframes == 24` or the state has returned to `seestate`, resets `castattacking`, `castframes`, and returns to `seestate`. Also handles the `S_PLAY_ATK1` gross hack via `goto stopattack`.
5. Sets `casttics` from the new state's `tics`; substitutes 15 if `tics == -1`.

**Returns:** void

---

### `F_CastResponder`

```c
boolean F_CastResponder(event_t* ev)
```

**Purpose:** Handles player key presses during the cast parade to kill the current cast member.

**Parameters:**
- `ev` — the input event

**Returns:** `true` if consumed (always, for any keydown event).

**Key logic:**
- Ignores non-keydown events.
- If `castdeath` is already true, returns immediately (animation already playing).
- On a keydown: sets `castdeath = true`, transitions to the monster's `deathstate`, plays its `deathsound`, resets `castframes` and `castattacking`.

---

### `F_CastPrint`

```c
void F_CastPrint(char* text)
```

**Purpose:** Renders a centered name label at y=180 during the cast parade.

**Parameters:**
- `text` — the name string to print

**Key logic:**
1. First pass: computes total pixel width of `text` using `hu_font` glyph widths.
2. Second pass: draws each glyph via `V_DrawPatch` starting at `cx = 160 - width/2` to center the text horizontally.

**Returns:** void

---

### `F_CastDrawer`

```c
void F_CastDrawer(void)
```

**Purpose:** Renders the cast parade frame: background, monster name, and current sprite.

**Key logic:**
1. Draws the `BOSSBACK` lump as the full-screen background via `V_DrawPatch`.
2. Calls `F_CastPrint` with the current cast member's name.
3. Looks up the sprite for `caststate`: indexes into `sprites[caststate->sprite].spriteframes[caststate->frame & FF_FRAMEMASK]`, retrieves lump index and flip flag from slot `[0]`.
4. Draws the patch at (160, 170); uses `V_DrawPatchFlipped` if `flip` is set.

**Returns:** void

---

### `F_DrawPatchCol`

```c
void F_DrawPatchCol(int x, patch_t* patch, int col)
```

**Purpose:** Draws a single column of a patch directly into `screens[0]`, used for the bunny scroll.

**Parameters:**
- `x` — horizontal pixel position on screen
- `patch` — pointer to the source patch
- `col` — column index within the patch

**Key logic:** Iterates over all posts of the column (`column->topdelta != 0xff`), copying source bytes vertically into `screens[0]` at the correct row positions (post's `topdelta` offset × `SCREENWIDTH`).

**Returns:** void

---

### `F_BunnyScroll`

```c
void F_BunnyScroll(void)
```

**Purpose:** Draws the episode 3 DOOM 1 ending: two wide patches (PFUB2 and PFUB1) scroll horizontally to reveal a bunny image, followed by an animated "END" logo.

**Key logic:**
1. Loads `PFUB2` (right half) and `PFUB1` (left half) at `PU_LEVEL` priority.
2. Computes `scrolled = 320 - (finalecount - 230) / 2`, clamped to [0, 320]. This drives the scroll position.
3. For each screen column x: if `x + scrolled < 320`, draws column `x + scrolled` of PFUB2; otherwise draws from PFUB1 with adjusted offset.
4. Before `finalecount == 1130`: returns (pure scroll, no logo).
5. At `finalecount` 1130–1179: draws `END0` centered.
6. After 1180: computes `stage = (finalecount - 1180) / 5`, clamped to [0, 6]. When `stage` increases, plays `sfx_pistol` and draws `END{stage}`.

**Static local:** `laststage` tracks the last displayed stage to avoid redundant sound triggers.

**Returns:** void

---

### `F_Drawer`

```c
void F_Drawer(void)
```

**Purpose:** Top-level draw dispatch for the finale, called each render frame when `gamestate == GS_FINALE`.

**Key logic:**
- `finalestage == 2`: calls `F_CastDrawer`.
- `finalestage == 0`: calls `F_TextWrite`.
- `finalestage == 1`: draws episode-specific art:
  - Episode 1, retail: `CREDIT`; episode 1 other: `HELP2`.
  - Episode 2: `VICTORY2`.
  - Episode 3: `F_BunnyScroll()`.
  - Episode 4: `ENDPIC`.

**Returns:** void

---

## Dependencies

| Module | Usage |
|--------|-------|
| `i_system.h` | `I_Error` (indirectly via game errors) |
| `m_swap.h` | `SHORT()` / `LONG()` macros for endian-safe patch field access |
| `z_zone.h` | Not directly called but used via WAD cache tags (`PU_LEVEL`, `PU_CACHE`) |
| `v_video.h` | `V_DrawPatch`, `V_DrawPatchFlipped`, `V_MarkRect`, `V_DrawBlock`, `screens[]` |
| `w_wad.h` | `W_CacheLumpName`, `W_CacheLumpNum` for loading flats, patches, and sprites |
| `s_sound.h` | `S_ChangeMusic`, `S_StartMusic`, `S_StartSound` |
| `dstrings.h` | `E1TEXT` … `T6TEXT`, `CC_*` cast name strings |
| `sounds.h` | `mus_*` and `sfx_*` enum values |
| `doomstat.h` | `gamemode`, `gameepisode`, `gamemap`, `players[]`, `MAXPLAYERS`, `gameaction`, `gamestate`, `viewactive`, `automapactive` |
| `r_state.h` | `sprites[]`, `firstspritelump` for cast sprite lookup |
| `hu_stuff.h` | `hu_font[]`, `HU_FONTSIZE`, `HU_FONTSTART` for text rendering |
| `f_wipe.c` | `wipegamestate` (extern) to force screen wipes on stage transitions |
| `info.h` (via `r_state.h`) | `states[]`, `mobjinfo[]` for cast animation state lookup |
