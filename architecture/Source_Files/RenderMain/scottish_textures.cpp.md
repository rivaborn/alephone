# Source_Files/RenderMain/scottish_textures.cpp

## File Purpose
Core software texture rasterizer for the Aleph One game engine. Implements perspective-correct texture mapping for polygons and rectangles using fixed-point math, with support for multiple transfer modes (textured, static, tinted, landscaped) and bit depths (8/16/32-bit). Named after a 1994 internal development comment ("this is not your father's texture mapping library").

## Core Responsibilities
- Allocate and manage DDA line tables and precalculation buffers for rasterization
- Rasterize textured polygons using both horizontal and vertical scanline orientations
- Rasterize clipped and scaled textured rectangles
- Precalculate perspective-correct texture coordinates and shading indices per polygon line/column
- Build Digital Differential Analyzer (DDA) edge tables for polygon boundary tracing
- Support multiple transfer modes and alpha blending strategies (fast/nice)
- Handle landscape-specific texture mapping with tiling and optional vertical repeating
- Dispatch rendering to appropriate bit-depth and alpha-mode template implementations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `polygon_definition` | struct | Input polygon with vertices, transfer mode, texture, origin, flags |
| `rectangle_definition` | struct | Input rectangle with bounds, clipping, texture, flip/transform flags |
| `bitmap_definition` | struct | Screen/texture buffer with row pointers, dimensions, bytes-per-row |
| `view_data` | struct | Camera parameters (FOV, projection, yaw/pitch/roll, depth intensity) |
| `_horizontal_polygon_line_data` | struct | Precalculated source texture coordinates and deltas per scanline |
| `_vertical_polygon_data` | struct | Header with downshift, x0, width for vertical polygon rendering |
| `_vertical_polygon_line_data` | struct | Precalculated texture_y, texture_dy, shading table, texture base per column |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `scratch_table0` | `short*` | static | First DDA line coordinate table (8192 entries) |
| `scratch_table1` | `short*` | static | Second DDA line coordinate table (8192 entries) |
| `precalculation_table` | `void*` | static | Buffer for per-line/column precalculated data |

## Key Functions / Methods

### allocate_texture_tables
- Signature: `void allocate_texture_tables()`
- Purpose: Allocate global scratch buffers needed for DDA and precalculation during rasterization
- Inputs: None
- Outputs/Return: None (allocates globals)
- Side effects: Allocates `scratch_table0`, `scratch_table1`, `precalculation_table` with `new[]`
- Calls: `new[]` (C++ allocation)
- Notes: Called once at engine startup; asserts all allocations succeed

### texture_horizontal_polygon
- Signature: `void Rasterizer_SW_Class::texture_horizontal_polygon(polygon_definition& textured_polygon)`
- Purpose: Rasterize a textured polygon using horizontal scanline order
- Inputs: Reference to `polygon_definition` with vertices, texture, transfer mode
- Outputs/Return: Draws to screen buffer
- Side effects: Modifies global `scratch_table0`/`scratch_table1`; dispatches to template rendering
- Calls: `build_x_table()`, `TEXBITS_DISPATCH` macros, `_pretexture_horizontal_polygon_lines`, landscape/texture rendering templates
- Notes: Returns early if transfer mode is `_static_transfer` (delegates to vertical mapper); validates vertex bounds; uses DDA edge tables to precalculate texture coords

### texture_vertical_polygon
- Signature: `void Rasterizer_SW_Class::texture_vertical_polygon(polygon_definition& textured_polygon)`
- Purpose: Rasterize a textured polygon using vertical scanline order
- Inputs: Reference to `polygon_definition`
- Outputs/Return: Draws to screen buffer
- Side effects: Modifies global scratch tables; precalculates per-column texture data
- Calls: `build_y_table()`, `_pretexture_vertical_polygon_lines`, texture/randomize/tint rendering templates
- Notes: Delegates `_big_landscaped_transfer` to horizontal mapper; validates vertex bounds

### texture_rectangle
- Signature: `void Rasterizer_SW_Class::texture_rectangle(rectangle_definition& textured_rectangle)`
- Purpose: Rasterize a textured rectangle with optional scaling, clipping, and horizontal flip
- Inputs: Reference to `rectangle_definition` with bounds, texture, transfer mode
- Outputs/Return: Draws to screen buffer
- Side effects: Populates precalculation table with per-column data; modifies scratch tables
- Calls: `calculate_shading_table()`, texture/randomize/tint rendering templates
- Notes: Handles clipping against screen and rectangle bounds; calculates texture scale from size ratio; special handling for texture "first/last" row markers (big-endian u16); supports horizontal mirroring

### build_x_table
- Signature: `static short *build_x_table(short *table, short x0, short y0, short x1, short y1)`
- Purpose: Build a DDA line table of x-coordinates from (x0,y0) to (x1,y1), one x per scanline
- Inputs: Destination table pointer, two endpoints
- Outputs/Return: Returns table pointer on success, NULL on invalid input
- Side effects: Fills table with x-coordinates
- Calls: `std::abs()`, `SGN()`, internal DDA iteration
- Notes: Returns NULL if `dy <= 0` (vertical or backwards line); uses Bresenham-style error term `d` for subpixel accuracy

### build_y_table
- Signature: `static short *build_y_table(short *table, short x0, short y0, short x1, short y1)`
- Purpose: Build a DDA line table of y-coordinates from (x0,y0) to (x1,y1), one y per vertical step
- Inputs: Destination table pointer, two endpoints
- Outputs/Return: Returns table pointer on success, NULL on invalid input
- Side effects: Fills table with y-coordinates (forward or backward depending on direction)
- Calls: `std::abs()`, `SGN()`, internal DDA iteration
- Notes: Returns NULL if `dx < 0`; handles both positive and negative y-direction fill

### calculate_shading_table
- Signature: `static void calculate_shading_table(void* &result, view_data *view, void *shading_tables, short depth, _fixed ambient_shade)`
- Purpose: Select the appropriate shading table index based on depth and ambient lighting, accounting for bit depth
- Inputs: View parameters, shading table base, depth (world units or clamped to 16-bit), ambient shade
- Outputs/Return: `result` (reference) set to pointer to selected shading table
- Side effects: None
- Calls: Macros `SHADE_TO_SHADING_TABLE_INDEX()`, `DEPTH_TO_SHADE()`, `PIN()`
- Notes: Handles negative ambient shade (full tint), calculates shade from depth and maximum intensity; uses switch on global `bit_depth`; contains fallback for unknown bit depths

### _pretexture_horizontal_polygon_lines
- Signature: `template<int TEXBITS> static void _pretexture_horizontal_polygon_lines(polygon_definition *polygon, bitmap_definition *screen, view_data *view, _horizontal_polygon_line_data *data, short y0, short *x0_table, short *x1_table, short line_count)`
- Purpose: Precalculate perspective-correct source texture coordinates for each scanline of a horizontal polygon
- Inputs: Polygon, tables of left/right x-coordinates per scanline, starting y, line count
- Outputs/Return: Fills `data` array with `source_x`, `source_y`, `source_dx`, `source_dy` (scaled by `HORIZONTAL_FREE_BITS`)
- Side effects: None
- Calls: `cosine_table[]`, `sine_table[]`, inline calculations
- Notes: Handles both high-precision (polygon near z=0) and standard precision paths; calculates shading table per line; performs DDA-style incremental update

### _pretexture_vertical_polygon_lines
- Signature: `template<int TEXBITS> static void _pretexture_vertical_polygon_lines(polygon_definition *polygon, bitmap_definition *screen, view_data *view, _vertical_polygon_data *data, short x0, short *y0_table, short *y1_table, short line_count)`
- Purpose: Precalculate perspective-correct source texture coordinates for each column of a vertical polygon
- Inputs: Polygon, tables of top/bottom y-coordinates per column, starting x, column count
- Outputs/Return: Fills `_vertical_polygon_line_data` array with `texture_y`, `texture_dy`, shading table, texture base per column
- Side effects: None
- Calls: Inline fixed-point and trigonometric calculations
- Notes: Handles reduction of large numerators/denominators to avoid overflow; uses "long-distance friendly" adjustments for large world_x values; computes world-space x from texture coordinate then back-maps to texture y

### _prelandscape_horizontal_polygon_lines
- Signature: `static void _prelandscape_horizontal_polygon_lines(polygon_definition *polygon, bitmap_definition *screen, view_data *view, _horizontal_polygon_line_data *data, short y0, short *x0_table, short *x1_table, short line_count)`
- Purpose: Precalculate landscape-specific texture coordinates with tiling and optional vertical repeating
- Inputs: Polygon (texture is landscape), view parameters with landscape_yaw, line tables, starting y, line count
- Outputs/Return: Fills `data` array with `source_x`, `source_dx`, `source_y` for landscape sampling
- Side effects: None (reads landscape options from view)
- Calls: `View_GetLandscapeOptions()`, `NextLowerExponent()`
- Notes: Uses separate horizontal and vertical pixel deltas; applies optional vertical repeat masking; landscape texture width determines repeat height via OpenGL aspect-ratio exponent

## Control Flow Notes
**Initialization**: `allocate_texture_tables()` must be called once before any rasterization.

**Per-polygon rendering flow**:
1. `texture_horizontal_polygon()` or `texture_vertical_polygon()` called (entry point)
2. Validate vertices and find extremal vertices (highest/lowest or leftmost/rightmost)
3. Build DDA edge tables by calling `build_x_table()` or `build_y_table()` for each polygon edge in order
4. Precalculate mode-specific data by calling `_pretexture_*_polygon_lines()` template
5. Dispatch to bit-depth and transfer-mode specific rendering template (e.g., `texture_horizontal_polygon_lines<pixel16, _sw_alpha_nice, 8>`)
6. Template in `low_level_textures.h` performs actual per-pixel sampling and blending

**Per-rectangle rendering flow**:
1. `texture_rectangle()` called
2. Clip bounds against screen and rectangle
3. Calculate texture scale and horizontal mirroring
4. For each visible column: calculate y-range, texture boundaries, shading table
5. Populate precalculation table with per-column data
6. Dispatch to rendering template

Not inferable from this file:
- How the templates in `low_level_textures.h` actually write pixels
- Memory layout of shading tables (managed elsewhere)
- Details of polygon/rectangle geometry preparation (vertex transformation)

## External Dependencies
- **Notable includes**: `cseries.h` (common types), `low_level_textures.h` (rendering templates and DDA helpers), `render.h` (view_data, polygon_definition), `Rasterizer_SW.h` (class definition), `preferences.h` (graphics_preferences global), `SW_Texture_Extras.h` (SW_Texture class for opacity)
- **Defined elsewhere**: 
  - `cosine_table[]`, `sine_table[]` ΓÇö trigonometric lookup
  - `TEXBITS_DISPATCH`, `TEXBITS_DISPATCH_2` ΓÇö macros that branch on texture size
  - `texture_horizontal_polygon_lines`, `texture_vertical_polygon_lines`, `landscape_horizontal_polygon_lines`, `randomize_vertical_polygon_lines`, `tint_vertical_polygon_lines` ΓÇö template functions
  - `View_GetLandscapeOptions()` ΓÇö returns landscape rendering options
  - `graphics_preferences` global ΓÇö alpha blending mode setting
  - Global `bit_depth` ΓÇö current rendering depth (8/16/32)
  - Global `screen` in Rasterizer_SW_Class ΓÇö destination bitmap
  - Global `view` in Rasterizer_SW_Class ΓÇö view parameters
