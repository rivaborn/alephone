# Source_Files/RenderMain/RenderRasterize_Shader.h

## File Purpose
Defines a shader-based GPU rendering rasterizer that extends the base RenderRasterizerClass to support modern OpenGL rendering with framebuffer objects, shader pipelines, and advanced visual effects. Serves as the GPU-accelerated rendering backend for Aleph One's geometry and sprite pipeline.

## Core Responsibilities
- Coordinate GPU-accelerated rendering of BSP tree geometry (floors, ceilings, walls, objects)
- Manage OpenGL framebuffer objects and shader rasterizer state setup
- Create and manage TextureManager instances for wall and sprite textures
- Implement clipping, viewport transforms, and endpoint storage for rasterization
- Render viewer-relative sprites (HUD elements, weapons, foreground objects) in rendering tree
- Apply per-surface visual effects (blur, weapon flare, self-luminosity pulsation)
- Override base rasterizer methods to delegate to shader-based implementations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| RenderRasterize_Shader | class | GPU-accelerated rasterizer extending RenderRasterizerClass |
| Blur | class | Visual blur effect (forward declared, owned via unique_ptr) |
| Rasterizer_Shader_Class | class | Shader pipeline backend rasterizer (pointer member) |

## Global / File-Static State
None.

## Key Functions / Methods

### RenderRasterize_Shader() / ~RenderRasterize_Shader()
- **Signature:** Constructor and destructor
- **Purpose:** Initialize/cleanup the shader rasterizer and associated resources (blur effect, FBO swap chains via Rasterizer_Shader_Class)
- **Side effects:** Manages lifetime of blur effect and rasterizer backend

### setupGL()
- **Signature:** `virtual void setupGL(Rasterizer_Shader_Class& Rasterizer)`
- **Purpose:** Initialize OpenGL state and bind the shader rasterizer backend before rendering
- **Inputs:** Reference to shader rasterizer instance
- **Side effects:** Configures GL context for this rendering pass

### render_tree()
- **Signature:** `virtual void render_tree(void)`
- **Purpose:** Main entry point; initiates rendering of the entire visible BSP tree
- **Calls:** Invokes protected virtual methods (render_node, render_node_floor_or_ceiling, etc.) for each visible surface
- **Notes:** Called once per frame; implements painter's algorithm by traversing BSP in visibility order

### renders_viewer_sprites_in_tree()
- **Signature:** `bool renders_viewer_sprites_in_tree() { return true; }`
- **Purpose:** Signals that this rasterizer handles viewer sprites (HUD/weapon) within the render tree, not separately
- **Return:** Always true for shader rasterizer

### setupWallTexture() / setupSpriteTexture()
- **Signature:** `std::unique_ptr<TextureManager> setupWallTexture(const shape_descriptor& Texture, short transferMode, float pulsate, float wobble, float intensity, float offset, RenderStep renderStep)`
- **Purpose:** Create and configure a TextureManager for wall or sprite surfaces with visual effects
- **Inputs:** Texture descriptor, blend/transfer mode, effect parameters (pulsate, wobble, intensity, offset), render pass
- **Outputs/Return:** Ownership-managed TextureManager instance
- **Side effects:** Allocates OpenGL textures and color tables

### render_node() [protected]
- **Signature:** `virtual void render_node(sorted_node_data *node, bool SeeThruLiquids, RenderStep renderStep)`
- **Purpose:** Render a single BSP node; dispatches to floor/ceiling/wall/object renderers as needed
- **Inputs:** Sorted node, transparency flags, render pass identifier

### render_node_floor_or_ceiling() / render_node_side() / render_node_object() [protected]
- **Purpose:** Render individual surface types (horizontal, vertical, object sprites) with clipping and texturing
- **Inputs:** Surface geometry, polygon/side data, void presence flags (for transparency), render pass
- **Notes:** Void-present flag suppresses semitransparency when rendering against empty space

### clip_to_window() [protected]
- **Purpose:** Apply viewport clipping constraints to upcoming rasterization
- **Inputs:** Clipping window definition

### render_viewer_sprite_layer() / render_viewer_sprite() [protected]
- **Purpose:** Render weapon sprite, HUD elements, and other viewer-relative objects after geometry
- **Inputs:** Render rectangle, render pass

## Control Flow Notes
1. **Initialization:** `setupGL()` binds the shader rasterizer backend at frame start
2. **Rendering:** `render_tree()` traverses BSP in visibility order, calling protected virtual methods
3. **Geometry pass:** For each visible node, `render_node()` dispatches to surface-specific renderers
4. **Texturing:** Surfaces call `setupWallTexture()` or `setupSpriteTexture()` to allocate and configure textures
5. **Effects:** Visual effects (blur, pulsate, wobble, flare) are applied per-surface via TextureManager parameters
6. **Foreground:** Viewer sprites rendered after all geometry via `render_viewer_sprite_layer()`
7. **GPU pipeline:** Rasterizer_Shader_Class handles actual GPU submission and framebuffer swapping

## External Dependencies
- **Base class:** RenderRasterizerClass (RenderRasterize.h) ΓÇö defines core rasterizer interface
- **Geometry types:** sorted_node_data, polygon_data, horizontal_surface_data, vertical_surface_data, endpoint_data, side_data (map.h)
- **Rendering types:** render_object_data, clipping_window_data, rectangle_definition, shape_descriptor (render.h, map.h)
- **GPU backend:** Rasterizer_Shader_Class (Rasterizer_Shader.h) ΓÇö shader pipeline implementation
- **Texture management:** TextureManager (OGL_Textures.h) ΓÇö handles OpenGL texture allocation and state
- **Framebuffer objects:** FBO, FBOSwapper (OGL_FBO.h) ΓÇö GPU render target management
- **Utilities:** Blur class (forward declared; implementation elsewhere), std::unique_ptr (memory)
- **Enums/Constants:** RenderStep (kDiffuse, kGlow pass identifiers), world_distance, long_vector2d, shape_descriptor
