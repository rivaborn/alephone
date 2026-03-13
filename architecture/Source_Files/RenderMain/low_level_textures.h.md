# Source_Files/RenderMain/low_level_textures.h

## File Purpose
Low-level software rasterizer for textured polygons in the Aleph One engine. Provides template-based texture mapping for horizontal and vertical screen-aligned polygons with support for multiple pixel formats, blending modes, and effects (tinting, randomization).

## Core Responsibilities
- Template-based texture rasterization for horizontal and vertical polygons
- Multi-mode pixel blending: off (opaque), fast (averaging), and nice (alpha-blended)
- Landscape texture mapping with dynamic width scaling
- Tinting and static/randomization effects for polygons
- Support for 8/16/32-bit pixel formats with transparency handling
- Per-pixel shading table lookup and color channel masking for alpha blending

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `texture_constants<TEXBITS>` | template class | Compile-time texture size constants (width, height, shift, fractional bits) for powers of 2 |
| `_horizontal_polygon_line_header` | struct | Metadata for horizontal texture mapping (y coordinate downshift) |
| `_horizontal_polygon_line_data` | struct | Per-line source texture coordinates and deltas (source_x/y, dx/dy) and shading table |
| `_vertical_polygon_data` | struct | Vertical polygon screen position and dimensions (x0, width, downshift) |
| `_vertical_polygon_line_data` | struct | Per-column texture data: shading table, texture pointer, texture y-coordinate and delta |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `texture_random_seed()` | `inline uint16&` | static (function-local) | Returns reference to static PRNG seed for randomization effects; persists across calls |

## Key Functions / Methods

### texture_horizontal_polygon_lines
- **Signature:** `template <typename T, int sw_alpha_blend, int TEXBITS> void texture_horizontal_polygon_lines(bitmap_definition *texture, bitmap_definition *screen, view_data *view, _horizontal_polygon_line_data *data, short y0, short *x0_table, short *x1_table, short line_count, uint8 *opacity_table = 0)`
- **Purpose:** Rasterize textured horizontal polygons by filling horizontal scanlines with perspective-corrected texture samples.
- **Inputs:** Texture and screen bitmaps; per-line data (source coords/deltas); screen position tables (x0/x1 per line); line count; optional opacity table
- **Outputs/Return:** Writes pixels directly to screen bitmap via row_addresses
- **Side effects:** Modifies screen bitmap memory; reads shading tables and texture memory
- **Calls:** `write_pixel<T, sw_alpha_blend, false>` per pixel; accesses SDL_PixelFormat for alpha-nice mode
- **Notes:** Innermost loop increments source_x/y per pixel; macros like `HORIZONTAL_HEIGHT_DOWNSHIFT` derive from TEXBITS template parameter

### landscape_horizontal_polygon_lines
- **Signature:** `template <typename T> void landscape_horizontal_polygon_lines(bitmap_definition *texture, bitmap_definition *screen, view_data *view, _horizontal_polygon_line_data *data, short y0, short *x0_table, short *x1_table, short line_count)`
- **Purpose:** Special-case horizontal rasterizer for landscape textures; no vertical perspective (only x-coordinate varies).
- **Inputs:** Texture/screen bitmaps; per-line data; screen position tables; line count
- **Outputs/Return:** Writes pixels to screen bitmap
- **Side effects:** Modifies screen; computes dynamic downshift from texture height at runtime
- **Calls:** Shading table lookup only; no blending
- **Notes:** Landscape texture width determined by `NextLowerExponent(texture->height)`; simpler than full perspective

### texture_vertical_polygon_lines
- **Signature:** `template <typename T, int sw_alpha_blend, bool check_transparent> void texture_vertical_polygon_lines(bitmap_definition *screen, view_data *view, _vertical_polygon_data *data, short *y0_table, short *y1_table, uint8 *opacity_table = 0)`
- **Purpose:** Rasterize textured vertical polygons by filling vertical strips with blended texture samples. Highly optimized with SIMD-like 4├ùcolumn parallel loop.
- **Inputs:** Screen bitmap; polygon data (x0, width, downshift); per-column y-range tables; optional opacity table
- **Outputs/Return:** Writes pixels to screen
- **Side effects:** Modifies screen; accesses SDL_PixelFormat for alpha-nice mode
- **Calls:** `write_pixel<T, sw_alpha_blend, check_transparent>` and `copy_check_transparent<T, check_transparent>`; reads texture and shading table memory
- **Notes:** Unrolled 4-column loop for cache efficiency when line_count ΓëÑ 4 and x-aligned; falls back to single-column for unaligned/short regions; comment indicates y0==screen->height case is invalid but unread

### tint_vertical_polygon_lines
- **Signature:** `template <typename T> void tint_vertical_polygon_lines(bitmap_definition *screen, view_data *view, _vertical_polygon_data *data, short *y0_table, short *y1_table, uint16 transfer_data)`
- **Purpose:** Apply tint/shading effects to vertical polygons by remapping pixels through per-pixel tint tables.
- **Inputs:** Screen bitmap; polygon data; y-range tables; transfer_data (low byte = tint table index)
- **Outputs/Return:** Modifies screen pixels in-place
- **Side effects:** Reads/writes screen and tint table memory; asserts tint_table_index is valid
- **Calls:** `tint_tables_pointer<T>` and `get_pixel_tint<T>` for each non-zero texture pixel
- **Notes:** Only processes pixels where texture is non-zero; uses `_fixed` for texture_y coordinate with downshift

### randomize_vertical_polygon_lines
- **Signature:** `template <typename T, bool check_transparent> void randomize_vertical_polygon_lines(bitmap_definition *screen, view_data *view, _vertical_polygon_data *data, short *y0_table, short *y1_table, uint16 transfer_data)`
- **Purpose:** Apply static/randomization effect to vertical polygons; randomly skips pixels based on PRNG seed.
- **Inputs:** Screen bitmap; polygon data; y-range tables; transfer_data (threshold for PRNG acceptance)
- **Outputs/Return:** Writes random values to screen pixels
- **Side effects:** Updates global `texture_random_seed()`; reads texture and screen memory
- **Calls:** `randomize_vertical_polygon_lines_write<T>` for accepted pixels; updates seed via Galois LFSR
- **Notes:** LFSR polynomial 0xb400 for seed shifting; seed threshold (`drop_less_than`) from transfer_data

### Helper Functions (summarized)
- `write_pixel`: Template dispatch on alpha blend mode; applies shading table ┬▒ fast averaging or alpha blending
- `copy_check_transparent`: Conditional shading table lookup respecting transparency bit
- `average`, `alpha_blend`: Template color blending for 16/32-bit pixels (bit-level ARGB/565 math)
- `get_pixel_tint`, `tint_tables_pointer`: Tint table color remapping (specialized per pixel depth)
- `NextLowerExponent`: Compute integer logΓéé by bit shifting

## Control Flow Notes
These functions are called during the polygon rendering phase of each screen refresh. Horizontal functions iterate y-major (top to bottom) filling horizontal scanlines; vertical functions iterate x-major (left to right) filling vertical strips. Both use precomputed shading tables for fast color lookup. The `TEXBITS_DISPATCH` macros (defined but not used in this file) suggest template instantiation is delegated to a caller based on texture size.

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_PixelFormat`, `SDL_PixelFormat::Rmask/Gmask/Bmask/Rshift/Gshift/Bshift/Rloss/Gloss/Bloss` for pixel format queries
- **cseries.h:** Basic types (`uint16`, `uint32`, `pixel8`, `pixel16`, `pixel32`, `_fixed`, `byte`), constants (`FIXED_FRACTIONAL_BITS`, `TRIG_SHIFT`, `WORLD_FRACTIONAL_BITS`), macros (`MAX`, `MIN`, `fc_assert`)
- **preferences.h:** Blend mode constants (`_sw_alpha_off`, `_sw_alpha_fast`, `_sw_alpha_nice`)
- **textures.h:** `bitmap_definition` structure (width, height, bytes_per_row, row_addresses, flags)
- **scottish_textures.h:** Polygon/tint structures (`_vertical_polygon_data`, `_vertical_polygon_line_data`, `tint_table8/16/32`), constants (`number_of_shading_tables`, `PIXEL8_MAXIMUM_COLORS`, `PIXEL16/32_MAXIMUM_COMPONENT`)
