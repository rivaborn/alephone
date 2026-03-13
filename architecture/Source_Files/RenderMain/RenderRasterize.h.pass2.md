# Source_Files/RenderMain/RenderRasterize.h - Enhanced Analysis

## Architectural Role

RenderRasterizerClass is the **coordinate transformation and geometry dispatcher** in the rendering pipelineΓÇöthe critical link between depth-sorted visibility trees (from RenderSortPolyClass) and platform-specific rasterizers (software, OpenGL, shader-based). It owns the tree traversal logic, applies multi-stage clipping (XY screen bounds ΓåÆ elevation Z ΓåÆ vertical walls), and delegates final pixel output via a polymorphic RasterizerClass backend (Strategy pattern). This two-pass design (Diffuse + Glow) enables basic separated lightingΓÇöcritical for Marathon's emissive surfaces and visual effects.

## Key Cross-References

### Incoming (who depends on this)
- **RenderMain/render.cpp** orchestrates per-frame calls: likely instantiates RenderRasterizerClass subclasses or calls render_tree() after visibility/sorting phases complete
- **RenderSortPolyClass** (via RSPtr member) provides the pre-built depth-ordered tree; this class only *executes* it, doesn't build it
- **RasterizerClass subclasses** (Rasterizer_SW, Rasterizer_OGL, Rasterizer_Shader) are called back via RasPtrΓåÆtexture_horizontal_polygon(), texture_vertical_polygon(), texture_rectangle()

### Outgoing (what depends on)
- **RenderSortPoly.h** ΓåÆ sorted_node_data, RenderSortPolyClass
- **RenderPlaceObjs.h** ΓåÆ render_object_data (projected sprite objects)
- **Rasterizer.h** ΓåÆ RasterizerClass interface (backend strategy)
- **world.h** ΓåÆ long_vector2d, world_distance (coordinate types), trigonometric utilities, long_point3d
- **render.h** ΓåÆ view_data (camera/viewport state), polygon_data, bitmap_definition, clipping_window_data, endpoint_data, horizontal_surface_data, side_texture_definition
- **Implicit**: render_object_data lifetime and projection state set by RenderPlaceObjs before this class traverses it

## Design Patterns & Rationale

### Strategy Pattern (Rasterizer Backend)
RasterizerClass (RasPtr) is swappable: software texturing (scottish_textures), OpenGL fixed-function, or shader-based. This class abstracts *tree traversal* from *pixel delivery*, allowing new rendering backends without rewriting geometry logicΓÇöessential for cross-platform support (Win32, macOS, Linux with OpenGL variants).

### Template Method (Two-Pass Rendering)
The public `render_tree()` likely calls protected `render_tree(RenderStep renderStep)` twice: once with kDiffuse (base textures), once with kGlow (emissive surfaces). This defers to subclasses and RasterizerClass to handle multi-layer compositionΓÇöcommon in late-90s/early-2000s real-time engines.

### Data-Driven Geometry
flagged_world_point2d/3d and vertical_surface_data structures bundle geometry + metadata (clipping flags, lighting, texture). This avoids deep object hierarchies and suits graphics pipelines that batch-process geometry by type (floors ΓåÆ ceilings ΓåÆ walls ΓåÆ sprites).

### Visitor-like Traversal
`render_tree(step)` ΓåÆ `render_node()` ΓåÆ `{render_node_floor_or_ceiling, render_node_side, render_node_object}` is a lightweight visitor: each polygon node's surfaces and objects are dispatched to specialized handlers, but the traversal itself stays in one class.

## Data Flow Through This File

```
Inputs (per frame):
  view_data* ΓåÆ camera position, orientation, projection matrix
  RSPtr (RenderSortPolyClass*) ΓåÆ pre-sorted tree of visible polygons + clipping windows
  RasPtr (RasterizerClass*) ΓåÆ backend rasterizer (software/OpenGL/shader)

Per render_tree() call:
  For each pass (kDiffuse, kGlow):
    Γö£ΓöÇ render_tree(step)
    Γöé  ΓööΓöÇ render_node(each visible polygon node, step)
    Γöé     Γö£ΓöÇ render_node_floor_or_ceiling(clipping_window, polygon, surface, void_present, step)
    Γöé     Γöé  ΓööΓöÇ RasPtrΓåÆtexture_horizontal_polygon(...)
    Γöé     Γö£ΓöÇ render_node_side(clipping_window, vertical_surface_data, void_present, step)
    Γöé     Γöé  ΓööΓöÇ xy/z/xz clip pipeline (transforms world ΓåÆ screen space)
    Γöé     Γöé  ΓööΓöÇ RasPtrΓåÆtexture_vertical_polygon(...)
    Γöé     ΓööΓöÇ render_node_object(render_object_data, other_side_of_media, step)
    Γöé        ΓööΓöÇ RasPtrΓåÆtexture_rectangle(...)

Outputs:
  Modified RasPtr internal state (framebuffer/VRAM updates) ΓÇö final composited frame
```

**Clipping Pipeline** (for walls):
1. `xz_clip_vertical_polygon()` ΓåÆ clips 3D vertices to screen XZ bounds
2. `xy_clip_horizontal_polygon()` ΓåÆ clips 2D floor vertices to screen XY bounds
3. `z_clip_horizontal_polygon()` ΓåÆ clips floor vertices to elevation range (h0, h1, hmax)
4. Helper: `store_endpoint()` caches transformed endpoints (likely for cache coherency across multiple wall segments sharing endpoints)

## Learning Notes

### Era-Specific Design (Year 2000)
- **No deferred rendering**: Immediate-mode dispatching to rasterizer; no accumulation buffers or compute passes
- **Fixed-function backend assumption**: RasterizerClass interface targets immediate-mode GPUs (OpenGL 1.2ΓÇô1.4); modern engines use compute/drawcalls
- **Clipping in CPU**: Polygon clipping done in C++ before dispatch, not in shaders; modern engines clip in vertex shaders or use compute
- **Long-distance refactoring comment**: "made this more long-distance friendly" (repeated 3├ù in comments) signals Marathon 1 had int16 coordinate limits; this refactor to long_vector2d/world_distance expanded playable spaceΓÇöa pragmatic fix not architectural redesign

### Marathon-Specific Rendering
- **Void boundary handling** (`void_present` flag): Suppresses semitransparency artifacts where game world meets "outside" void. This is a visual quality tweak for an engine where BSP walkthrough is common.
- **Media-side aware sprites** (`other_side_of_media`): Indicates liquid surface awareness at sprite depth-sort time. Monsters/items on the opposite side of water from player need special Z-orderingΓÇötight coupling between physics (media containment) and rendering (depth sort).
- **No dynamic shadows**: Multi-pass rendering is for diffuse+glow, not for shadow accumulation.

## Potential Issues

1. **Virtual method overhead on inner loop**: Every `render_node*` call is virtual; polymorphic dispatch per polygon could be slow. Mitigated if polygon counts are modest (~100ΓÇô1000 visible polys in Marathon), but no hot-path optimization visible.

2. **Mutable member pointers, no lifetime management**: view, RSPtr, RasPtr are bare pointers set externally. Caller must guarantee they're valid for the entire render_tree() call. No assertions or null checks visible.

3. **No error handling**: Clipping functions return vertex counts; out-of-bounds clipping or degenerate geometry silently produces 0 vertices, rendering nothing instead of erroring.

4. **Implicit state in Rasterizer backend**: The clipping window (screen bounds, elevation ranges) is calculated by RenderSortPoly and passed here, but the rasterizer state (current texture, transfer mode, lighting) is managed by RasPtr. Tight coupling between this class and the rasterizer's internal state machine.

5. **`renders_viewer_sprites_in_tree()` virtual hook is underexplained**: Probably separates first-person weapon rendering (HUD layer) from world layer, but subclasses and the calling site are not obvious from this header alone.

---

**Key Takeaway**: This is a classic early-2000s rendering dispatcherΓÇöclean separation of geometry logic from pixel backend, supporting multiple rendering paths (software/OpenGL/shader) without duplication. The "Marathon-isms" (void handling, media boundaries) and "long-distance" comments reveal the file evolved from a more limited earlier design. The data-driven structure (flagged_world_point2d, vertical_surface_data) and multi-pass approach are idiomatic for engines of that era.
