# File Overview

`d_textur.h` is a minimal header that defines the `pic_t` structure — a simple unmasked block of pixels used to represent flat (uncompressed) graphic resources. It exists as an isolation header to separate texture-related type definitions from the rest of the engine, making it easier to use this type in modules that do not need to import the full rendering pipeline headers.

In practice, `pic_t` represents raw pixel bitmaps that are not column-based (unlike the `patch_t` format used for most sprites and wall textures). The comment "Flats?" suggests this was intended for floor/ceiling texture data, though the actual flat rendering in DOOM uses raw byte arrays rather than this structure.

## Global Variables

This header defines no global variables.

## Functions

This header declares no functions.

## Data Structures

### `pic_t` (struct)

A simple descriptor for an unmasked rectangular block of pixel data.

```c
typedef struct {
    byte width;   // Width of the image in pixels.
    byte height;  // Height of the image in pixels.
    byte data;    // First byte of the pixel data (used as an array anchor).
} pic_t;
```

**Usage pattern:** `data` is declared as a single `byte` but is used as the beginning of a variable-length pixel buffer that extends beyond the struct. This is a common C idiom for variable-length structures. The actual pixel data is accessed by taking the address of `data` and treating it as a `byte*` array of `width * height` bytes.

**Note:** This structure is not heavily used in the linuxdoom codebase. The raw flat textures are typically accessed directly as byte arrays from the WAD lump cache. The structure appears to be a forward-looking or legacy type definition kept for completeness.

## Dependencies

| File | Reason |
|------|--------|
| `doomtype.h` | Provides the `byte` typedef (`unsigned char`). |
