# Source_Files/RenderMain/textures.cpp - Enhanced Analysis

## Architectural Role

This file is a **critical low-level bitmap utility module** supporting the rendering pipeline's texture initialization phase. It bridges the Files subsystem (which loads binary shape collections from WAD archives) and the RenderMain rasterization backends (which consume pre-computed row address tables). The file's three core functions orchestrate bitmap memory layout, enabling efficient per-row pixel access without repeated format scanning during per-frame rendering. It also provides palette remapping support for color-space transformations (e.g., infravision tinting, suit damage overlay).

## Key Cross-References

### Incoming (who depends on this)
- **shapes.cpp** (RenderMain) ΓÇö Calls `precalculate_bitmap_row_addresses()` when loading Marathon shape collections from WAD files; exports bitmap_definition structs to renderer
- **OGL_Textures.h/cpp** (RenderMain) ΓÇö OpenGL texture lifecycle management; likely calls these functions when uploading textures to VRAM or applying color tables
- **scottish_textures.cpp** (RenderMain) ΓÇö Software rasterizer; consumes cached `row_addresses[]` arrays populated by `precalculate_bitmap_row_addresses()`
- **Rasterizer_Shader.cpp** (RenderMain) ΓÇö GPU-based rasterizer; may call `remap_bitmap()` for palette effects before uploading to shader
- **AnimatedTextures.h/cpp** (RenderMain) ΓÇö Animated wall textures; may use row address tables for frame sequencing
- **textures.h** (public header) ΓÇö Exports these functions to entire engine; `calculate_bitmap_origin()` is used during bitmap_definition initialization

### Outgoing (what this file depends on)
- **cseries.h** ΓåÆ Type definitions (`pixel8`, `byte`, `int32`, `uint16`), endianness detection
- **textures.h** ΓåÆ bitmap_definition struct declaration, `NONE` constant
- No runtime dependencies on other subsystems (pure low-level utility)

## Design Patterns & Rationale

1. **Pointer Arithmetic for Memory Layout** ΓÇö `calculate_bitmap_origin()` uses raw offset math to skip past the bitmap_definition struct and row pointer array. This avoids malloc overhead and keeps bitmap metadata and pixel data cache-coherent in a single allocation.

2. **Dual RLE Format Support via Conditional Compilation** ΓÇö MARATHON1 (16-bit run counts) vs MARATHON2 (big-endian first/last offsets) suggests the engine evolved across two game releases. Compile-time selection rather than runtime branching indicates performance was critical; modern engines would use a version tag in the binary format.

3. **Big-Endian Manual Parsing in MARATHON2** ΓÇö The explicit `<< 8 | lower_byte` pattern for reading big-endian uint16 metadata indicates **original Mac development** (Motorola 68k, PowerPC were big-endian). This is preserved in the cross-platform rewrite to maintain binary format compatibility with original Marathon 2 WADs.

4. **Precalculation + Lazy Access Pattern** ΓÇö `precalculate_bitmap_row_addresses()` caches row pointers **once at load** rather than computing them repeatedly during rendering. This trades upfront scan cost for O(1) per-row access during rasterization.

5. **Lookup Table Remapping** ΓÇö `remap_bitmap()` + `map_bytes()` use a 256-entry color lookup table for palette transformation. This is 1990s-era technique; modern engines use shader-based color correction. The pattern is economical for 8-bit paletted graphics but doesn't scale to modern 32-bit color.

## Data Flow Through This File

```
[WAD File loaded by Files subsystem]
    Γåô
[bitmap_definition struct created + raw pixel data buffered]
    Γåô
calculate_bitmap_origin()  ΓåÉ Compute start of pixel data (skip struct + row pointers)
    Γåô
precalculate_bitmap_row_addresses()  ΓåÉ Cache row start addresses
    Γöé  (scans RLE metadata if compressed)
    Γöé
[Optional: palette change or color effect triggered]
    Γåô
remap_bitmap()  ΓåÉ Apply 256-entry color LUT to all pixels
    Γöé  (scans rows via cached row_addresses)
    Γöé
[Bitmap ready for rendering]
    Γåô
scottish_textures.cpp or Shader backend consumes row_addresses[]
    Γåô
Per-frame rasterization uses cached row pointers ΓåÆ O(1) row lookup
```

**State transitions:** Uninitialized ΓåÆ Origin calculated ΓåÆ Row addresses cached ΓåÆ (optionally remapped) ΓåÆ Ready for rendering. Each step must occur in order; calling `remap_bitmap()` before `precalculate_bitmap_row_addresses()` causes undefined behavior (row_addresses not yet populated).

## Learning Notes

**What studying this file teaches:**

1. **1990s Texture Architecture** ΓÇö Paletted 8-bit pixels, RLE compression, manual endianness handling. Modern engines use compressed formats (ASTC, BC1-7), GPU-side tiling, and mipmap hierarchies.

2. **Binary Format Preservation** ΓÇö The MARATHON2 RLE variant's big-endian byte layout was **never changed** despite porting to x86 (little-endian). This indicates the engine prioritized **WAD compatibility** over cleanupΓÇöa pragmatic tradeoff for a community-driven port.

3. **Layout Flexibility via Flags** ΓÇö The `_COLUMN_ORDER_BIT` flag allows row-major and column-major storage in the same code path. This suggests different texture types (walls vs sprites) may have used different layouts for cache efficiency or compression benefits.

4. **Absence of Validation** ΓÇö No checks for malformed RLE runs, invalid row counts, or buffer overruns. This reflects an era where debug builds and developer foresight were the primary safety mechanisms.

## Potential Issues

1. **RLE Underflow on Corrupted Data** (MARATHON2, line ~100)
   ```cpp
   uint16 last = *row_address++ << 8 | *row_address++;
   row_address += last - first;  // Unsigned underflow if last < first
   ```
   If a WAD contains a malformed RLE row with `last < first`, the subtraction underflows, causing `row_address` to jump billions of bytes forward. Subsequent row parsing corrupts memory.

2. **Palette Index Bounds** (`map_bytes`, line ~135)
   ```cpp
   *buffer = table[*buffer];  // Assumes table has 256 entries
   ```
   No validation that `*buffer < 256`. For 8-bit pixels this is safe, but if called with other data types or corrupt buffers, out-of-bounds table access occurs.

3. **Missing Precalculation Check** ΓÇö `remap_bitmap()` assumes `precalculate_bitmap_row_addresses()` was called first. Calling in wrong order produces garbage (row_addresses array not initialized). No sentinel or validation.

4. **Format Ambiguity** ΓÇö The `bytes_per_row != NONE` check doesn't distinguish between future unsupported compression schemes. If a new format (e.g., deflate-compressed) is added without incrementing a version field, the code silently falls through to MARATHON2 code, causing misinterpretation.
