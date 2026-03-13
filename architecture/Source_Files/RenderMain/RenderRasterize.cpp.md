# Source_Files/RenderMain/RenderRasterize.cpp

## File Purpose
Implements polygon clipping and rasterization for the Aleph One game engine. Processes a sorted tree of world geometry and clips polygons to viewport boundaries, converting them to screen-space textured polygons for rendering. Handles complex cases like semi-transparent liquids, flooded platforms, and animated textures.

## Core Responsibilities
- Tree traversal: iterates sorted polygons from back-to-front
- Polygon clipping: clips horizontal and vertical polygons to viewport boundaries using Sutherland-Hodgman algorithm
- Coordinate transformation: converts world coordinates to screen-space with perspective division
- Surface rendering: dispatches floors, ceilings, walls, and objects to rasterizer
- Liquid media handling: manages rendering of see-through vs opaque liquid surfaces
- Texture animation translation: applies animated texture frame selection
- Void detection: tracks whether polygon boundaries face empty space (for transparency rendering)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `flagged_world_point2d` | struct | 2D world point with per-vertex clipping boundary flags |
| `flagged_world_point3d` | struct | 3D world point with clipping flags (for vertical geometry) |
| `vertical_surface_data` | struct | Wall/side descriptor: height bounds, endpoints, texture, ambient lighting |
| `horizontal_surface_data` | struct | Floor/ceiling descriptor: height, origin, texture, lightsource (from map.h) |
| `polygon_definition` | struct | Output polygon for rasterizer: vertices, texture, shading, transfer mode (from render.h) |
| `RenderStep` | enum | Rendering pass selector: `kDiffuse` or `kGlow` |

## Global / File-Static State
None file-static; state is instance variables in `RenderRasterizerClass`:

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `view` | `view_data*` | instance | Camera/viewport state |
| `RSPtr` | `RenderSortPolyClass*` | instance | Sorted polygon tree (contains `SortedNodes` vector) |
| `RasPtr` | `RasterizerClass*` | instance | Backend rasterizer; receives textured polygons for final rendering |

## Key Functions / Methods

### render_tree()
- Signature: `void render_tree()` / `void render_tree(RenderStep renderStep)`
- Purpose: Main entry point; traverses the sorted polygon tree and renders all visible geometry back-to-front
- Inputs: Optional `renderStep` (default: `kDiffuse`)
- Outputs/Return: None
- Side effects: Iterates `RSPtr->SortedNodes`, calls `render_node()` for each; queries `graphics_preferences` and OpenGL config for liquid transparency
- Calls: `render_node()` for each sorted node
- Notes: Determines whether liquids are semi-transparent based on graphics mode (OpenGL vs software alpha blending)

### render_node()
- Signature: `void render_node(sorted_node_data *node, bool SeeThruLiquids, RenderStep renderStep)`
- Purpose: Renders a single polygon: its walls, floors, ceilings, and associated objects
- Inputs: Sorted node (polygon index + clipping windows), liquid transparency flag, render step
- Outputs/Return: None
- Side effects: Calls render_node_side, render_node_floor_or_ceiling, render_node_object; queries platform, polygon, media, endpoint, line, side accessors
- Calls: `get_polygon_data()`, `get_platform_data()`, `find_flooding_polygon()`, `get_media_data()`, `get_endpoint_data()`, `get_line_data()`, `get_side_data()`, `render_node_floor_or_ceiling()`, `render_node_side()`, `render_node_object()`
- Notes: Handles flooded platform floor disguise; manages liquid boundary clipping; distinguishes above/below media for sprite rendering order

### render_node_floor_or_ceiling()
- Signature: `void render_node_floor_or_ceiling(clipping_window_data *window, polygon_data *polygon, horizontal_surface_data *surface, bool void_present, bool ceil, RenderStep renderStep)`
- Purpose: Clips and renders a horizontal surface (floor or ceiling)
- Inputs: Viewport window, polygon geometry, surface properties (height, texture, etc.), void-presence flag, ceiling indicator
- Outputs/Return: None
- Side effects: Allocates textured_polygon on stack; calls clipping functions and RasPtr->texture_horizontal_polygon(); performs perspective division
- Calls: `AnimTxtr_Translate()`, `xy_clip_horizontal_polygon()` (left/right bounds), `z_clip_horizontal_polygon()` (top/bottom bounds), `get_endpoint_data()`, `get_shape_bitmap_and_shading_table()`, `get_light_intensity()`, `instantiate_polygon_transfer_mode()`, `RasPtr->texture_horizontal_polygon()`
- Notes: Reverses vertex order for ceiling polygons; handles division-by-zero in perspective transform; applies transfer mode scaling (2x/4x)

### render_node_side()
- Signature: `void render_node_side(clipping_window_data *window, vertical_surface_data *surface, bool void_present, RenderStep renderStep)`
- Purpose: Clips and renders a wall (vertical surface)
- Inputs: Viewport window, wall surface, void-presence flag
- Outputs/Return: None
- Side effects: Stack-allocates polygon_definition and temporary vertex arrays; clips to viewport; applies texture scaling
- Calls: `AnimTxtr_Translate()`, `xy_clip_line()` (clip endpoints to X bounds), `xz_clip_vertical_polygon()` (clip to Z bounds), `get_shape_bitmap_and_shading_table()`, `get_light_intensity()`, `instantiate_polygon_transfer_mode()`, `RasPtr->texture_vertical_polygon()`
- Notes: Constructs a trapezoid from two endpoints; handles transfer mode scaling and texture origin calculations; includes division-by-zero guards for perspective transform

### render_node_object()
- Signature: `void render_node_object(render_object_data *object, bool other_side_of_media, RenderStep renderStep)`
- Purpose: Renders sprite/object with liquid surface clipping
- Inputs: Object data, media-boundary-side flag
- Outputs/Return: None
- Side effects: Modifies object->rectangle clipping bounds; calls RasPtr->texture_rectangle()
- Calls: `RasPtr->texture_rectangle()`
- Notes: Uses XOR of view boundary and media-side flag to determine clip direction; sets BelowLiquid flag for 3D models

### xy_clip_horizontal_polygon()
- Signature: `short xy_clip_horizontal_polygon(flagged_world_point2d *vertices, short vertex_count, long_vector2d *line, uint16 flag)`
- Purpose: Clips a horizontal (floor/ceiling) polygon to one viewport boundary line using Sutherland-Hodgman algorithm
- Inputs: Vertex array, count, clipping line (normal vector), flag to mark clipped vertices
- Outputs/Return: New vertex count after clipping
- Side effects: Modifies vertex array in-place (may reorder/insert vertices); uses memmove for compaction
- Calls: `xy_clip_flagged_world_points()`, memmove()
- Notes: State machine tracks in/out transitions; uses cross product to test vertex side; handles vertex-on-line case; manages memory carefully to avoid overflows

### z_clip_horizontal_polygon()
- Signature: `short z_clip_horizontal_polygon(flagged_world_point2d *vertices, short vertex_count, long_vector2d *line, world_distance height, uint16 flag)`
- Purpose: Clips horizontal polygon in the height (Z) dimension at a fixed Y-coordinate height
- Inputs: Vertex array (2D points), count, clipping line (X component), world height, flag
- Outputs/Return: New vertex count
- Side effects: Modifies vertex array in-place
- Calls: `z_clip_flagged_world_points()`, memmove()
- Notes: Similar structure to xy_clip but works in XY plane at fixed height; used for viewport top/bottom bounds

### xz_clip_vertical_polygon()
- Signature: `short xz_clip_vertical_polygon(flagged_world_point3d *vertices, short vertex_count, long_vector2d *line, uint16 flag)`
- Purpose: Clips vertical (wall) polygon to viewport XZ boundary plane
- Inputs: 3D vertex array, count, clipping line normal, flag
- Outputs/Return: New vertex count
- Side effects: In-place vertex array modification
- Calls: `xz_clip_flagged_world_points()`, memmove()
- Notes: 3D version of Sutherland-Hodgman; vertices may be CW or CCW on screen

### xy_clip_flagged_world_points(), z_clip_flagged_world_points(), xz_clip_flagged_world_points()
- Purpose: Compute intersection of line segment against clipping line using linear interpolation
- Notes: All use fixed-point arithmetic with shift logic to avoid overflow; compute parameter t and interpolate along edge; preserve endpoint flags

### xy_clip_line(), store_endpoint()
- Purpose: Specialized helpersΓÇöxy_clip_line clips a 2-point line segment; store_endpoint converts endpoint coordinates (handling long-distance transformations)
- Notes: Minor utility functions

## Control Flow Notes
**Render pipeline:**
1. `render_tree()` (entry point) ΓåÆ iterates `RSPtr->SortedNodes`
2. For each node ΓåÆ `render_node()` 
3. `render_node()` processes floors, walls, ceilings, objects in sequence
4. Each surface ΓåÆ `render_node_floor_or_ceiling()` / `render_node_side()` / `render_node_object()`
5. Surface functions ΓåÆ clip with `*_clip_*()` functions (chained left/right, then top/bottom)
6. After clipping ΓåÆ transform to screen-space, build `polygon_definition`, dispatch to `RasPtr`

**Clipping strategy:** Successive clipping passes: XY bounds (viewport left/right), then Z/height bounds (top/bottom). Sutherland-Hodgman state machine tracks polygon entry/exit.

## External Dependencies
- **Game world data:** `polygon_data`, `side_data`, `line_data`, `endpoint_data`, `platform_data` (map.h); `media_data` (media.h); `light_data`, `get_light_intensity()` (lightsource.h)
- **Rendering:** `RasterizerClass`, `RenderSortPolyClass`, `polygon_definition`, `view_data` (render.h, Rasterizer.h, RenderSortPoly.h)
- **Textures:** `AnimTxtr_Translate()` (AnimatedTextures.h); `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()` (render.h)
- **Configuration:** `OGL_ConfigureData`, `Get_OGL_ConfigureData()`, `OGL_Flag_LiqSeeThru` (OGL_Setup.h); `graphics_preferences`, `screen_mode` (preferences.h, screen.h)
- **Platform:** `PLATFORM_IS_FLOODED()`, `find_flooding_polygon()` (platforms.h, map.h)
