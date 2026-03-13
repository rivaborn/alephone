# Source_Files/RenderMain/RenderRasterize.h

## File Purpose
Defines RenderRasterizerClass, which converts a sorted visibility tree and placed objects into screen-space geometry for rasterization. Acts as the bridge between world-space rendering data and platform-specific rasterizers (software or OpenGL), handling clipping, lighting, and coordinate transforms.

## Core Responsibilities
- Traverse sorted polygon tree (RenderSortPolyClass) in depth order
- Rasterize floors, ceilings, and walls with visibility clipping windows
- Render sprite objects with proper depth ordering and media-boundary semantics
- Apply geometric clipping (XY screen bounds, Z elevation, XZ vertical bounds)
- Delegate final pixel output to RasterizerClass backend
- Support multi-pass rendering (diffuse + glow layers)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `flagged_world_point2d` | struct | 2D point with clipping flags for floor/ceiling vertices |
| `flagged_world_point3d` | struct | 3D point with clipping flags for wall vertices |
| `vertical_surface_data` | struct | Wall surface: geometry, lighting delta, texture, height bounds |
| `RenderStep` | enum | Rendering pass (diffuse=solid, glow=emissive) |
| `RenderRasterizerClass` | class | Main traversal and coordination class |

## Global / File-Static State
None.

## Key Methods

### render_tree (public entry point)
- Signature: `virtual void render_tree()`
- Purpose: Public rendering entry point; initiates all render passes
- Inputs: none
- Outputs/Return: void
- Side effects: Calls protected render_tree(RenderStep) for each pass
- Notes: Typically called once per frame from render.c

### render_tree (protected worker)
- Signature: `virtual void render_tree(RenderStep renderStep)`
- Purpose: Recursively traverse visibility tree and render a single pass
- Inputs: `renderStep` ΓÇö diffuse or glow layer
- Outputs/Return: void
- Side effects: Calls render_node() for each visible polygon; updates RasPtr
- Calls (direct): render_node()
- Notes: Depth-first traversal; RenderSortPolyClass has already computed visibility and depth order

### render_node
- Signature: `virtual void render_node(sorted_node_data *node, bool SeeThruLiquids, RenderStep renderStep)`
- Purpose: Render one polygon node: its floor/ceiling, walls, and contained objects
- Inputs: sorted node with polygon and object lists; transparency flag; render pass
- Outputs/Return: void
- Side effects: Calls render_node_floor_or_ceiling, render_node_side, render_node_object
- Calls (direct): render_node_floor_or_ceiling, render_node_side, render_node_object

### render_node_floor_or_ceiling
- Signature: `virtual void render_node_floor_or_ceiling(clipping_window_data *window, polygon_data *polygon, horizontal_surface_data *surface, bool void_present, bool ceil, RenderStep renderStep)`
- Purpose: Rasterize a single floor or ceiling surface
- Inputs: clipping window; polygon/surface definitions; void/ceiling flags; render pass
- Outputs/Return: void
- Side effects: Calls RasPtrΓåÆtexture_horizontal_polygon()
- Notes: `void_present` suppresses semitransparency artifacts at void boundaries

### render_node_side
- Signature: `virtual void render_node_side(clipping_window_data *window, vertical_surface_data *surface, bool void_present, RenderStep renderStep)`
- Purpose: Rasterize a wall (vertical surface with lighting and height bounds)
- Inputs: clipping window; wall surface data; void-border flag; render pass
- Outputs/Return: void
- Side effects: Calls RasPtrΓåÆtexture_vertical_polygon()

### render_node_object
- Signature: `virtual void render_node_object(render_object_data *object, bool other_side_of_media, RenderStep renderStep)`
- Purpose: Rasterize a sprite object (monster, item, etc.)
- Inputs: render object with rectangle and clipping; media-boundary flag; render pass
- Outputs/Return: void
- Side effects: Calls RasPtrΓåÆtexture_rectangle()
- Notes: `other_side_of_media` indicates sprite is on opposite side of liquid surface from viewer

### renders_viewer_sprites_in_tree
- Signature: `virtual bool renders_viewer_sprites_in_tree()`
- Purpose: Query whether this rasterizer draws viewer sprites (weapons) in main tree
- Inputs: none
- Outputs/Return: bool (default false)
- Notes: Allows subclasses to signal special sprite handling

## Notes
- **Clipping helpers** (xy_clip_horizontal_polygon, z_clip_horizontal_polygon, xz_clip_vertical_polygon, and their _flagged_world_points variants): Geometric clipping to screen bounds and elevation limits; return clipped vertex counts.
- **store_endpoint**: Transforms world-space endpoint to screen coordinates; marks endpoint as transformed.
- **xy_clip_line**: Clips line segments to screen-space XY bounds.
- Member pointers (view, RSPtr, RasPtr) are set before rendering; provide access to view state, sorted polygons, and the rasterizer backend.

## Control Flow Notes
**Per-frame flow**: render_tree() ΓåÆ render_tree(kDiffuse) ΓåÆ [depth-first tree walk] ΓåÆ render_node() ├ù N ΓåÆ {render_node_floor_or_ceiling, render_node_side, render_node_object} ΓåÆ RasterizerClass methods. Then repeat for kGlow pass. Visibility culling and depth sort are pre-computed by RenderVisTreeClass and RenderSortPolyClass; this class only executes the final traversal and delegates rasterization.

## External Dependencies
- `<vector>` ΓÇö STL
- `world.h` ΓÇö long_vector2d, world_distance, angle, long_point3d
- `render.h` ΓÇö view_data, polygon_data, bitmap_definition, render flags
- `RenderSortPoly.h` ΓÇö sorted_node_data, RenderSortPolyClass
- `RenderPlaceObjs.h` ΓÇö render_object_data
- `Rasterizer.h` ΓÇö RasterizerClass (abstract base for platform-specific rendering)
- **Defined elsewhere**: endpoint_data, horizontal_surface_data, clipping_window_data, line_clip_data, rectangle_definition, side_texture_definition
