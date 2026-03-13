# Source_Files/RenderMain/low_level_textures.h - Enhanced Analysis

## Architectural Role

This file implements the innermost loop of the software rasterizerΓÇöconverting texture coordinates into shaded pixels at the per-pixel level. It sits at the tail end of the RenderMain pipeline: **RenderVisTree** ΓåÆ **RenderSortPoly** ΓåÆ **RenderRasterize** ΓåÆ **scottish_textures** ΓåÆ **low_level_textures.h**. The templates here are instantiated by dispatcher functions in `scottish_textures.cpp` (such as `_pretexture_horizontal_polygon_lines` and `_pretexture_vertical_polygon_lines`), which handle coordinate setup and table generation. This file owns only the innermost sampling, blending, and writing logic.

## Key Cross-References

### Incoming (who depends on this file)
- **scottish_textures.cpp** ΓÇö Dispatcher functions instantiate templates here; pass precomputed shading tables, texture coordinates, and screen bitmap pointers
- **RenderRasterize.h/cpp** ΓÇö Upstream: computes clipping windows and screen-space coordinates for polygons; triggers scottish_textures calls
- **Rasterizer_SW.h** ΓÇö Software rasterizer abstraction; delegates polygon fill to scottish_textures (which uses this file)

### Outgoing (what this file depends on)
- **preferences.h** ΓÇö Reads `_sw_alpha_off`, `_sw_alpha_fast`, `_sw_alpha_nice` blend mode constants; branched per template instantiation (no runtime cost)
- **textures.h** ΓÇö Reads `bitmap_definition` (width, height, row_addresses array for screen and texture bitmaps)
- **SDL2 (SDL_PixelFormat)** ΓÇö External global `world_pixels` surface accessed during alpha-nice blending to extract color masks (Rmask, Gmask, Bmask)
- **cseries.h** ΓÇö Base types, macros (MAX, MIN), and fixed-point constants (FIXED_FRACTIONAL_BITS, TRIG_SHIFT, WORLD_FRACTIONAL_BITS)

## Design Patterns & Rationale

### 1. **Compile-Time Polymorphism via Templates**
Template parameters (`T` for pixel type, `sw_alpha_blend` for blend mode, `TEXBITS`/`check_transparent` for flags) encode **all algorithmic variants** at compile time, eliminating per-pixel branch misses. A modern engine would use SIMD intrinsics; this code targets early-2000s CPUs where branch prediction penalties dominated.

### 2. **Macro-Based Template Dispatch**
`TEXBITS_DISPATCH` macros (e.g., dispatch on texture.width Γêê {256, 512, 1024}) avoid if-chains by selecting the correct `TEXBITS` template parameter. This pattern reflects the engine's legacy: textures are powers of 2, so shifts replace expensive divisions.

### 3. **Fixed-Point Arithmetic Everywhere**
- `source_x`, `source_y` are 32-bit fixed-point coordinates
- `HORIZONTAL_WIDTH_DOWNSHIFT` (e.g., 32ΓêÆTEXBITS) shifts the integer part into position for direct array indexing
- No floating-point conversion; operations like `source_x >> HORIZONTAL_WIDTH_DOWNSHIFT` extract the texture texel in O(1)
- Rationale: Avoid FPU stalls on late-1990s PowerPC hardware (comment mentions "PPC might not write-allocate")

### 4. **4-Column Unrolled Loop in Vertical Renderer**
`texture_vertical_polygon_lines` has three phases:
- **Sync**: Find ymax across 4 columns
- **Parallel map**: Fill synchronized rows (all 4 columns advance in lockstep)
- **Desync**: Fill remaining uneven regions

Trades code size (~4├ù) for cache efficiency: reduces pointer arithmetic and row-address lookups by 4├ù. Comment "/* parallel map (x4) */" shows this was deliberate optimization for 1990s cache hierarchies.

### 5. **Transparency as Non-Branch Optimization**
`check_transparent` template parameter allows the compiler to eliminate `if (pixel != 0)` checks at compile time for opaque textures. Modern engines use dynamic branching; this reflects era when template instantiation was cheaper than branch misprediction.

## Data Flow Through This File

```
Input (from scottish_textures):
  - texture: bitmap_definition (8-bit indexed pixels, row_addresses)
  - screen: bitmap_definition (8/16/32-bit framebuffer, row_addresses)
  - shading_table: per-scanline color remap (T* pointing into scottish_textures' tables)
  - source_x, source_y: texture coordinates (fixed-point)
  - source_dx, source_dy: per-pixel deltas (╬ö in texture space)

Processing (per-pixel loop, innermost loop):
  1. Extract texture texel: base_address[ (source_y >> downshift) & mask ] + (source_x >> downshift)
  2. Remap via shading_table[texel]: indexed ΓåÆ 8/16/32-bit color
  3. Blend (if needed):
     - _sw_alpha_off: write directly
     - _sw_alpha_fast: average(shaded, existing pixel) ΓÇö cheap, lossy
     - _sw_alpha_nice: alpha_blend(shaded, existing, opacity_table[texel])
  4. Write to screen->row_addresses[y] + x

Output:
  - Modified framebuffer (pixels written directly to SDL_Surface memory)
```

The **shading table indirection** is key: it's a 256-entry lookup table (for 8-bit textures) that remaps palette indices ΓåÆ final RGB colors, baking in lighting, tinting, and effects.

## Learning Notes

### Idioms of This Era
1. **Avoiding Division**: Macro definitions like `HORIZONTAL_WIDTH_DOWNSHIFT = 32 - TEXBITS` transform `texture_x / 256` into `texture_x >> 5`, saved cycles on hardware without fast division.
2. **Preventing Cache Misses**: The 4-column unrolled loop and comments about rowbytes (704 vs. 640) show hardware-level awareness. Modern engines abstract this; this code reflects Bungie's optimization for specific PowerPC cache line sizes.
3. **Indexed Color Authority**: Texture lookups go through shading tables, not direct RGB. This allowed dynamic tinting (damage flashes, lighting), invisibility effects, and motion sickness wobbles without modifying VRAM.

### What Modern Engines Do Differently
- **Instancing + Shaders**: Modern rasterizers instance single fragment shader 100M times/frame; this code manually unrolled loops for cache.
- **Texture Caching**: GPU VRAM management is implicit; here, row_addresses are precomputed CPU pointers (fragile if memory moves).
- **Alpha Blending**: Modern: per-pixel alpha channel; here: optional opacity_table workaround, assuming specific formats.

## Potential Issues

1. **SDL_PixelFormat Lookup Per-Polygon**: `world_pixels->format` accessed during `_sw_alpha_nice` setup but **not cached**. If rendering thousands of textured polygons, this repeats the dereference. Modern: precompute at format selection time.

2. **Bit Masking Assumes Format**:
   - `pixel32 average()` assumes ARGB: `0xfffefefeL` hardcodes bit positions for 8888 layout
   - `pixel16 average()` assumes 565: `0xf7deU` hardcodes positions
   - If format is 5551, 4444, or BGR, the math produces garbage.

3. **Edge Case in Vertical Renderer**:
   Comment: `write = (T *)(screen->row_addresses[0] + bytes_per_row * y0) + x; // invalid but unread if y0 == screen->height`
   - Suggests unvalidated assumptions about y0 bounds; code relies on never reaching the write if out-of-bounds.

4. **LFSR Seed Limited to 16 Bits**:
   `texture_random_seed()` returns a static `uint16` seed, shifting via `0xb400` polynomial. With only 65K states, randomization effects may show visible patterns in large areas.

5. **Texture Coordinate Overflow**:
   Fixed-point coordinates (`source_x`, `source_y`) are `uint32`. If a polygon spans > 2^32 texels (with subpixel precision), wraps silently.

6. **Opacity Table Not Optional at Runtime**:
   `_sw_alpha_nice` blending branches on `opacity_table` being non-null, but the template assumes it's either always provided or always NULL. Passing NULL when `sw_alpha_blend == _sw_alpha_nice` causes dereference crash.
