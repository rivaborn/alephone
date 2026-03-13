# Source_Files/RenderMain/Rasterizer.h - Enhanced Analysis

## Architectural Role
This abstract base class is the **strategy interface** for the rendering pipeline's rasterization layer, enabling Aleph One to support multiple rendering backends (software, OpenGL 1.x, and modern shader-based) without coupling the visibility/sorting/placement pipeline to any specific implementation. The `render.h/cpp` orchestration layer calls into Rasterizer subclasses at the final stage of the per-frame render loop, after visibility determination and polygon sorting are complete.

## Key Cross-References

### Incoming (who depends on this file)
- **render.cpp** (`RenderMain/render.h/cpp`) ΓÇö main orchestration layer instantiates a concrete `RasterizerClass` subclass and calls `SetView()`, `Begin()`, the `texture_*()` methods, `SetForeground()`, and `End()` in sequence per frame
- **RenderVisTree.cpp** ΓÇö builds visibility tree and passes visible polygons to rasterizer
- **RenderPlaceObjs.h** ΓÇö projects and culls game objects, dispatches to rasterizer for rendering
- **RenderRasterize.h** ΓÇö coordinate transformation and polygon clipping before rasterization

### Outgoing (what this file depends on)
- **render.h** ΓÇö imports `view_data` (camera: position, orientation, FOV, viewport dimensions), `polygon_definition` (geometry + texture), `rectangle_definition` (sprite bounds + texture)
- **OGL_Render.h** ΓÇö conditionally included; OpenGL-specific declarations used by `Rasterizer_OGL` subclass only

## Design Patterns & Rationale

**Strategy Pattern**: Three concrete subclass strategies exist:
- `Rasterizer_SW` (Scottish Textures software rasterizer) ΓÇö CPU-based, no GPU required
- `Rasterizer_OGL` (OpenGL 1.x fixed-function adapter) ΓÇö legacy GPUs
- `Rasterizer_Shader` (modern GLSL shader-based) ΓÇö current-gen rendering with FBOs and post-processing

Selected at compile/runtime based on user preference or GPU capability, allowing single codebase to support 1990sΓÇô2020s hardware.

**Separation of Concerns**: View setup (`SetView`) is decoupled from rendering. Allows camera configuration to be computed once in `render.cpp` before dispatching geometry to the backend.

**Two-Pass Rendering**: `SetForeground()` + `SetForegroundView()` switch rendering mode after `End()`, suggesting the engine renders world geometry first, then overlays first-person weapon models and HUDΓÇöa common pattern in early FPS engines.

**Dual Polygon Orientation**: `texture_horizontal_polygon()` vs. `texture_vertical_polygon()` hints at an **optimization decision**: walls/floors are typically axis-aligned, so separate implementations allow backends to specialize (e.g., software rasterizer unrolls different inner loops, OpenGL sets different rasterization rules). Modern engines treat all polygons uniformly; Aleph One's split reflects 2000s-era per-polygon micro-optimization thinking.

## Data Flow Through This File

```
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé render.cpp (orchestrator)               Γöé
Γö£ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
Γöé SetView(view_data)                      Γöé  ΓåÉ Camera: position, orientation, FOV
Γöé  ΓööΓöÇ Backend computes screen-to-world    Γöé
Γöé     transform, initializes viewport     Γöé
Γö£ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
Γöé Begin()                                 Γöé  ΓåÉ Backend clears buffers, binds state
Γö£ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
Γöé ΓöîΓöÇ For each visible polygon:            Γöé
Γöé Γö£ΓöÇ texture_horizontal_polygon(poly)     Γöé  ΓåÉ Walls, floors: rasterize + texture map
Γöé ΓööΓöÇ texture_vertical_polygon(poly)       Γöé     Ceilings: rasterize + texture map
Γö£ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
Γöé ΓöîΓöÇ For each sprite/object:              Γöé
Γöé ΓööΓöÇ texture_rectangle(rect)              Γöé  ΓåÉ Sprites, HUD elements
Γö£ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
Γöé SetForeground()                         Γöé  ΓåÉ Switch to weapon/overlay mode
Γöé SetForegroundView(reflected)            Γöé  ΓåÉ Configure foreground camera/mirror
Γöé ΓöîΓöÇ For foreground objects:              Γöé
Γöé ΓööΓöÇ texture_rectangle(weapon)            Γöé  ΓåÉ Render first-person weapon in hand
Γö£ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
Γöé End()                                   Γöé  ΓåÉ Backend finalizes: swap buffers, etc.
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
         Γåô
    Framebuffer / Display
```

## Learning Notes

**Era-Specific Design**: This file exemplifies early-2000s graphics abstraction. Modern engines (Unity, Godot, Unreal) use scene graphs and material systems; Aleph One directly passes geometry to a rasterizer. The explicit state machine (`SetView` ΓåÆ `Begin` ΓåÆ geometry ΓåÆ `SetForeground` ΓåÆ `End`) is manual scheduling rather than declarative scene description.

**Foreground/Background Architecture**: The separate `SetForeground()` workflow reveals first-person FPS design assumptions baked into the rendererΓÇöweapons are always rendered in a fixed-position overlay layer. This is less flexible than modern skeletal animation + shader systems, but simple and efficient for era-appropriate hardware.

**Polygon Specialization**: The `horizontal_polygon` vs. `vertical_polygon` split is idiomatic to Aleph One's **grid-aligned level design** (borrowed from Marathon 2, which reused closed 2.5D geometry). Modern engines support arbitrary polygon orientation; Aleph One optimizes for the axis-aligned case.

## Potential Issues

- **Empty implementations**: Virtual methods with empty bodies (`{}`) mean subclasses must override *all* methods, even if a backend doesn't use them (e.g., `SetForeground()` for non-FPS uses). No safety mechanism flags a forgotten override at compile time.
- **No error semantics**: No return codes or exceptions; backends fail silently on VRAM exhaustion, shader compilation error, etc. Errors must be debugged via logging.
- **Tight coupling to coordinate system**: `SetView(view_data&)` assumes `view_data` fully describes the projection; changing coordinate systems requires modifying all subclasses.
- **Hardware abstraction leakage**: Conditional `#include "OGL_Render.h"` exposes OpenGL implementation details at the header level, rather than isolating them.
