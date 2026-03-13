# Source_Files/RenderMain/textures.h - Enhanced Analysis

## Architectural Role
This file bridges texture **storage** and **rasterization** within the rendering pipeline. It defines the low-level bitmap metadata structure used by all three rasterization backends (software, OpenGL, shader-based), and provides utilities for bitmap initialization (row address pre-computation) and pixel remapping (color/palette transformations). The `bitmap_definition_buffer` RAII class ensures safe allocation of the variable-length bitmap + row pointer array, a C idiom that is central to texture memory layout throughout the engine.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderMain/shapes.cpp** ΓÇô Loads shape collections (Marathon resources) containing bitmaps, calls `precalculate_bitmap_row_addresses()` during shape initialization
- **RenderMain/scottish_textures.cpp** ΓÇô Software rasterizer; accesses `row_addresses[]` for sequential scanline texture mapping via DDA tables
- **RenderMain/OGL_Textures.cpp** ΓÇô OpenGL backend; converts bitmap rows to GPU texture uploads
- **RenderMain/AnimatedTextures.cpp** ΓÇô Animated wall textures; coordinates bitmap remapping for frame sequencing
- **RenderMain/RenderRasterize.cpp** ΓÇô Pipeline orchestrator; queries bitmap metadata for clipping window and perspective transforms

### Outgoing (what this file depends on)
- **CSeries/cseries.h** ΓÇô `pixel8`, `byte`, `int16` type definitions; foundational platform abstraction
- **C++ `<vector>`** ΓÇô `std::vector<byte>` backing for `bitmap_definition_buffer`
- **RenderMain/textures.cpp** ΓÇô Implementation of declared functions; actual row address calculation and remapping logic

## Design Patterns & Rationale

### Variable-Length Struct Array Pattern
```c
struct bitmap_definition {
  // ... fields ...
  pixel8 *row_addresses[1];  // Actually points to [row_count] elements
};
```
This is a **flexible array member idiom** (C99 standard, widely used pre-C99 in this codebase). Memory layout: `[bitmap_definition header (30 bytes)] + [row_count-1 additional pointers]`. Why? **Cache efficiency**: Bitmap metadata and row pointer table are contiguous in one allocation, minimizing pointer chasing during rendering. Modern engines use separate arrays, but this pattern was common in the 1990sΓÇô2000s when memory and CPU cache were more constrained.

### RAII Wrapper Over C Struct
`bitmap_definition_buffer` provides:
- **Safe allocation**: Constructor computes exact byte count needed
- **Nullable state**: `empty()` check prevents null pointer dereference in `get()`
- **Assertion in constructor**: `assert(row_count >= 1)` enforces that at least one row pointer exists (the array must be non-empty for valid bitmaps)

Why not just allocate raw? **Prevents resource leaks** (destructor via `std::vector`) and makes ownership explicit in modern C++.

### Lookup Table Remapping
`map_bytes()` and `remap_bitmap()` are simple **palette/color indirection**. Why? 
- **Low-cost palette swaps**: Remap pixels without allocating new bitmap (useful for damage flash, invisibility, special effects)
- **8-bit indexed color idiom**: Engine uses 256-color indexed bitmaps (one byte per pixel), not direct RGBΓÇöefficient for memory and transformation tables fit in CPU L1 cache

## Data Flow Through This File

```
Shape File (WAD/Resource)
    Γåô
Load Bitmap Data
    Γåô
bitmap_definition_buffer allocated (header + row pointers)
    Γåô
precalculate_bitmap_row_addresses()  [populate row_addresses[] array]
    Γåô
Rasterizers (scottish_textures, OGL_Textures)
    Γö£ΓöÇΓåÆ Row-by-row texture mapping via row_addresses[y]
    ΓööΓöÇΓåÆ Per-scanline DDA interpolation for perspective correction
    
(Optional) Visual Effects
    Γåô
remap_bitmap() ΓåÆ map_bytes()  [apply color LUT]
    Γåô
Pixel data transformed in-place
```

## Learning Notes

**What this file teaches:**
1. **Pre-computed lookup tables**: Modern engines compute row addresses once, reuse infinitely (O(1) scanline access). This pattern appears throughout the engine (e.g., flood-map pathfinding, shading tables in `shapes.cpp`).
2. **Flat memory model**: Assumes pixel data follows metadata contiguously. This is an optimization for cache locality; complex modern renderers split this.
3. **Palette-based color**: 8-bit indexed bitmaps were standard in the Marathon era (1994ΓÇô2001). Modern engines use direct RGB/RGBA, but this codebase's choice enabled fast color transformations and small texture footprints.
4. **C struct flexibility**: Uses pointer-to-first-element trick to extend array size; requires manual memory computation (seen in the constructor's `row_count - 1` offset).

**Idiomatic vs. Modern:**
- **This era**: Bitmap and row pointers co-allocated; remapping via lookup tables.
- **Modern engines**: Separate texture metadata from pixel data; GPU handles palette/transformations via shaders.

## Potential Issues

1. **Unsafe pointer arithmetic in constructor**: 
   ```cpp
   buf(sizeof(bitmap_definition) + (row_count - 1) * sizeof(pixel8*))
   ```
   If `row_count == 0`, this allocates `sizeof(bitmap_definition) - sizeof(pixel8*)`, which is invalid. The assert prevents this in debug builds, but release builds silently allow a malformed buffer.

2. **No validation in callers**:
   `precalculate_bitmap_row_addresses()` assumes `bytes_per_row > 0`, `height > 0`, and `row_addresses[0]` points to valid pixel data. If any precondition is violated, memory corruption or segfault ensues.

3. **Mutable bitmap data**: `remap_bitmap()` and `map_bytes()` modify pixel data in-place without versioning. If two visual effects (e.g., invisibility + damage flash) try to remap simultaneously, results are undefined.

4. **Incomplete const-correctness**: `calculate_bitmap_origin()` takes non-const `bitmap_definition*`, though it should be const (only reads structure).
