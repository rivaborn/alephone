# Source_Files/RenderMain/RenderRasterize_Shader.h - Enhanced Analysis

## Architectural Role

This file defines the GPU-accelerated rasterization backend, acting as a **rendering strategy implementation** within the pluggable rasterizer architecture. It bridges the high-level rendering pipeline (visibility tree, polygon sorting, object placement) with the shader-based GPU subsystem (Rasterizer_Shader_Class), enabling modern multi-pass rendering with per-surface visual effects (pulsation, wobble, flare, luminosity). The class participates in a **Strategy pattern**: the renderer selects between RenderRasterize_SW (software), RenderRasterize_OGL (classic OpenGL), or this shader variant at runtime based on capabilities and user preferences.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderMain subsystem** (render.cpp): Creates/destroys RenderRasterize_Shader instances; calls render_tree() each frame
- **Rasterizer selection logic** (likely in RenderRasterize.cpp or render.cpp): Instantiates this class when shader backend is available
- **OGL_Render.h/cpp** and **OGL_Setup.h/cpp**: Initialize OpenGL context before setupGL() is called; provide texture loading infrastructure
- **AnimatedTextures.h/cpp**: Supplies animated texture frame indices to setupWallTexture()

### Outgoing (what this file depends on)
- **RenderRasterizerClass** (RenderRasterize.h): Defines virtual interface contract (render_tree, render_node, etc.)
- **Rasterizer_Shader_Class** (Rasterizer_Shader.h/cpp): GPU submission, FBO swapping, shader program management; **raw pointer member** (RasPtr) suggests lifetime managed externally, likely by parent render context
- **Map geometry types** (map.h): sorted_node_data, polygon_data, horizontal_surface_data, vertical_surface_data, endpoint_data, clipping_window_data
- **TextureManager** (OGL_Textures.h): Created on-demand for each visible wall/sprite surface; manages VRAM lifetime
- **Blur** (forward declared, implementation elsewhere): Manages blur post-processing; owned via unique_ptr
- **RenderStep enum**: Coordinates multi-pass rendering (likely kDiffuse, kGlow, etc.) defined in render.h or similar

## Design Patterns & Rationale

| Pattern | Implementation | Rationale |
|---------|---|---|
| **Strategy** | RenderRasterize_Shader implements RenderRasterizerClass virtual interface | Allows render.cpp to swap rasterizers (SW Γåö OGL Γåö Shader) without recompilation; enables fallback if GPU unavailable |
| **Template Method** | render_tree() orchestrates; render_node() dispatches to render_node_floor_or_ceiling, render_node_side, render_node_object | Separates algorithm structure (BSP traversal) from surface-specific rendering details |
| **Resource Acquisition Is Initialization (RAII)** | std::unique_ptr<Blur> and returned unique_ptr<TextureManager> | Automatic cleanup on scope exit; prevents resource leaks in complex control flow |
| **Lazy Initialization** | setupWallTexture() and setupSpriteTexture() called on-demand per surface | Avoids pre-allocating textures for invisible surfaces; reduces VRAM waste |
| **Delegation** | Actual GPU rasterization delegated to Rasterizer_Shader_Class | Separates high-level geometry logic from low-level GPU API details (GLSL compilation, VAO/VBO setup, FBO binding) |

**Architectural Tradeoff**: Raw pointer `Rasterizer_Shader_Class *RasPtr` is not owned (no unique_ptr) because lifetime is controlled by the parent render context (see render.cpp setup). This trades safety for flexibility: allows re-binding a different shader context mid-frame if needed (though likely not done in practice). Modern code would use reference or ensure clear ownership semantics.

## Data Flow Through This File

```
Frame Start (render.cpp):
  setupGL(rasterizer_backend) 
    ΓåÆ bind GPU context, validate capabilities
  
render_tree() [main entry]:
  ΓåÆ for each visible BSP node (from RenderVisTree):
    
    render_node(sorted_node) 
      ΓåÆ dispatches by surface type:
        
        Horizontal (floor/ceiling):
          render_node_floor_or_ceiling(polygon, surface)
            ΓåÆ clip_to_window(clipping)  [viewport constraint]
            ΓåÆ setupWallTexture(descriptor, effects...)
              [allocate VRAM texture, encode pulsate/wobble into UV shader params]
            ΓåÆ Rasterizer_Shader_Class::rasterize_polygon()
              [submit GPU draw call]
        
        Vertical (walls):
          render_node_side(surface)
            ΓåÆ setupWallTexture(...)
            ΓåÆ Rasterizer_Shader_Class::rasterize_polygon()
        
        Objects (sprites):
          render_node_object(render_object)
            ΓåÆ _render_node_object_helper()
            ΓåÆ setupSpriteTexture(rect, offset)
              [position relative to weapon viewport]
            ΓåÆ Rasterizer_Shader_Class::rasterize_sprite()
  
  render_viewer_sprite_layer()
    ΓåÆ render_viewer_sprite(rect) per HUD/weapon
      [foreground layer after all geometry]

Frame End:
  Blur::draw() [post-processing]
    [applies blur to FBO if active]
  Rasterizer_Shader_Class::swap_framebuffers()
    [ping-pong FBO ΓåÆ screen]
```

**Key observation**: Texture setup is **per-surface**, not per-material. This enables tight per-pixel control (wobble offset varies by polygon) and multi-pass rendering (render diffuse pass, then glow pass with different textures).

## Learning Notes

1. **Multi-pass rendering architecture**: The `RenderStep` parameter in method signatures suggests this class is designed for deferred/forward+ style rendering where geometry is rasterized multiple times with different shaders (e.g., kDiffuse for color, kGlow for self-luminous surfaces, kShadow for silhouettes). This is a significant evolution from fixed-function OpenGL.

2. **Effect parameterization**: pulsate, wobble, intensity, offset are passed to setupWallTexture/setupSpriteTexture rather than baked into geometry. This suggests effects are **shader-driven**: the GPU computes animated texture offsets and blending in real time rather than pre-computing all frames.

3. **Viewer-relative rendering**: `render_viewer_sprite()` and `render_viewer_sprite_layer()` handle weapons, HUD, foreground objects in screen-space, separate from world-space geometry. This decoupling allows independent animation and scaling of weapon sprites.

4. **Era-specific design** (early-2010s GPU rendering):
   - Uses OpenGL 2.1ΓÇô3.x era patterns (separate texture objects, manual uniform binding)
   - No compute shaders or indirect rendering (modern optimization)
   - Assumes immediate-mode vertex submission via Rasterizer_Shader_Class (likely not instanced)
   - FBO ping-pong for post-processing (blur) is relatively novel for Marathon engine; software rasterizer had no equivalent

5. **VRAM lifetime management**: TextureManager returned from setupWallTexture() is managed by caller (likely a frame-local pool); suggests textures are cached/reused across frame or freed after rendering. This is more sophisticated than naive per-frame allocation.

## Potential Issues

1. **Raw pointer ownership ambiguity**: `Rasterizer_Shader_Class *RasPtr` is not owned by this class. If the external shader context is destroyed while this rasterizer is in use, or if multiple RenderRasterize_Shader instances share one RasPtr, behavior is undefined. Should be documented or use reference_wrapper for clarity.

2. **Clipping window lifetime**: `clipping_window_data *win` passed to clip_to_window() ΓÇö if allocated on stack in render_node(), lifetime is safe; but if clip_to_window() caches the pointer, post-frame use could access freed memory. Implementation in .cpp file would clarify.

3. **Effect composition complexity**: Multiple independent effect parameters (pulsate, wobble, intensity, offset) passed to setupWallTexture() could lead to **shader explosion** ΓÇö if each combination requires a different shader variant, permutation explosion occurs. Modern engines use uber-shaders with branches; unclear how this engine handles it.

4. **Multi-pass overhead**: RenderStep parameter suggests surfaces are rendered multiple times. If not batched carefully, this could thrash GPU pipeline. No visible LOD or surface culling at the rasterizer level (culling happens upstream in RenderPlaceObjs).
