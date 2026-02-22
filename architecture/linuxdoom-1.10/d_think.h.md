# File Overview

`d_think.h` defines the foundational thinker subsystem for DOOM's game simulation. It provides the function-pointer types and the doubly-linked list node (`thinker_t`) that every active game object (map objects, moving sectors, lights, doors, platforms, etc.) uses to receive per-tick processing.

This is one of the most fundamental architectural headers in the engine. The "thinker" concept is DOOM's equivalent of an update/tick callback: any object that needs to run logic every game tic is added to the global thinker list maintained in `p_tick.c`. Each tic, `P_Ticker` walks the entire list and invokes each thinker's `function` pointer.

The design allows heterogeneous objects (players, monsters, projectiles, sector specials) to all participate in the same simulation loop with type-erased callbacks, enabling a simple and extensible architecture without requiring C++ virtual dispatch.

## Global Variables

This header defines no global variables. The thinker list head is a `thinker_t` defined in `p_tick.c`.

## Functions

No functions are declared in this header.

## Data Structures

### Action Function Pointer Types

Three function pointer typedefs cover the different calling conventions used for action callbacks:

```c
typedef void (*actionf_v)();            // No arguments.
typedef void (*actionf_p1)(void*);      // One generic pointer argument.
typedef void (*actionf_p2)(void*, void*); // Two generic pointer arguments.
```

- `actionf_v` is used for state machine action functions that take no parameters (e.g., `A_WeaponReady`).
- `actionf_p1` is the most common; it takes a single `void*` which is cast to `mobj_t*` or `player_t*` within the function. Nearly all monster AI and thinker functions use this signature.
- `actionf_p2` takes two generic pointers; used less frequently for functions needing two context objects.

---

### `actionf_t` (union)

A union holding any of the three function pointer types, allowing a single field to store any action callback regardless of signature:

```c
typedef union {
    actionf_p1 acp1;  // Single-argument action function.
    actionf_v  acv;   // No-argument action function.
    actionf_p2 acp2;  // Two-argument action function.
} actionf_t;
```

This union is used in `state_t` (actor state machine frames) and in `thinker_t`. The caller must know which member to use based on context.

---

### `think_t` (typedef)

```c
typedef actionf_t think_t;
```

A semantic alias for `actionf_t` specifically in the context of thinker callbacks. Using `think_t` in `thinker_t` documents intent while keeping the types identical.

---

### `thinker_t` (struct)

The core thinker node, embedded at the beginning of every object that participates in the per-tick update loop.

```c
typedef struct thinker_s {
    struct thinker_s* prev;     // Previous thinker in the doubly-linked list.
    struct thinker_s* next;     // Next thinker in the doubly-linked list.
    think_t           function; // The callback invoked each tic.
} thinker_t;
```

**Design pattern:** Because `thinker_t` is always the first field in structures like `mobj_t`, a pointer to any such object can be safely cast to `thinker_t*` and back. The thinker list in `p_tick.c` stores raw `thinker_t*` pointers, while the action functions cast them to the appropriate concrete type.

**Lifecycle:**
- Objects are added to the list by `P_AddThinker` (in `p_tick.c`).
- Objects are removed by setting `function.acv = (actionf_v)P_RemoveThinkerDelayed` or immediately by `P_RemoveThinker`.
- When an object's function pointer is set to `-1` (cast), it signals the thinker is being removed.

**Concurrency note:** The thinker list is singly-iterable during a tic — objects must not be freed mid-iteration; instead they are flagged and cleaned up between tics.

## Dependencies

| File | Reason |
|------|--------|
| (none directly) | This header is self-contained. It uses only built-in C types and the compiler's function pointer syntax. |

Headers that include `d_think.h`:
- `p_mobj.h` — `mobj_t` embeds `thinker_t` as its first member.
- `p_spec.h` — Various sector special structs (ceiling, floor, door, platform, etc.) embed `thinker_t`.
- `p_tick.h` — Declares the thinker list head and `P_Ticker`.
