# Source_Files/RenderMain/render.cpp - Enhanced Analysis

## Architectural Role

This file functions as the **render orchestrator and integration hub** for the entire rendering subsystem. Beyond basic entry-point duties, it choreographs visibility determination (RenderVisTree), polygon depth-sorting (RenderSortPoly), object projection (RenderPlaceObjs), and backend-agnostic rasterization (Rasterizer_SW/OGL/Shader) through a carefully layered pipeline. It bridges game state (weapons, shapes, models, camera) into render-ready structures and applies transformation effects (wobbles, fades, camera shake) that compose cleanly with any rasterizer backend. The file is also the integration point between the render and gameplay subsystemsΓÇöit pulls weapon data from `weapons.h`, shape collections from `interface.h`, model data from `OGL_Render.h`, and spatial state from the GameWorld/map subsystem.

## Key Cross-References

### Incoming (who depends on this file)
- **Shell/Interface lifecycle** ΓåÆ calls `render_view()` each frame as part of the main render loop
- **Weapon/HUD system** ΓåÆ provides player weapon display via `render_viewer_sprite_layer()`  
- **GameWorld** ΓåÆ passes camera position/orientation via `view_data` struct
- **Platform-specific shell code** ΓåÆ calls `allocate_render_memory()` at map startup
- **Marathon 1 exploration** (background) ΓåÆ uses `check_m1_exploration()` to mark unseen polygons
- **Preferences/configuration** ΓåÆ reads `graphics_preferences->screen_mode.acceleration` to select GPU vs. CPU rendering

### Outgoing (what this file depends on)
- **RenderVisTree/RenderSortPoly/RenderPlaceObjs/RenderRasterize** ΓåÆ Four modular render-stage classes that this file instantiates and cross-links
- **Rasterizer backends (Rasterizer_SW, Rasterizer_OGL, Rasterizer_Shader)** ΓåÆ Runtime-selected rendering implementation
- **Weapons subsystem** (`weapons.h`) ΓåÆ fetches weapon display info via `get_weapon_display_information()`
- **Shape/texture system** (`interface.h`) ΓåÆ accesses shape bitmaps, collections, and shading tables
- **Map geometry** (`map.h`) ΓåÆ reads polygon/line/endpoint data for visibility and media boundary checks
- **Light subsystem** (`lightsource.h`) ΓåÆ fetches global shading tables for transfer mode tinting
- **Media/liquid system** (`media.h`) ΓåÆ checks liquid boundaries for render visibility and camera effects
- **Player state** (`player.h`) ΓåÆ reads player position, weapons, and view parameters
- **OpenGL subsystem** (`OGL_Render.h`) ΓåÆ conditionally queries 3D model replacements for weapons
- **AnimatedTextures** ΓåÆ updates animated wall texture frames
- **Dynamic limits** ΓåÆ reads max geometry sizes for pre-allocation

## Design Patterns & Rationale

**Orchestrator/Director Pattern**: `render_view()` doesn't perform visibility or sorting itselfΓÇöit delegates to specialized classes and wires them together (e.g., `RenderSortPoly.RVPtr = &RenderVisTree`). This decouples stages and allows independent optimization/modification.

**Runtime Backend Strategy Selection**: The file selects between software, OpenGL, and shader rasterizers at render-time by checking `graphics_preferences`. This avoids compilation variants and allows dynamic switchingΓÇöuseful for debugging or quality-per-framerate scaling.

**Transfer Modes as Pre-Rasterization Modulation**: Effects (wobbles, slides, fades, shrinks) are applied to polygon/rectangle *data* via `instantiate_polygon_transfer_mode()` and `instantiate_rectangle_transfer_mode()` *before* rasterization. This decouples effects from any specific rasterizer implementation, allowing the same effect to work identically on CPU or GPU backends.

**Parallel Exploration Tree**: `check_m1_exploration()` exploits the visibility tree's side effect (marking explored polygons) without rendering. It swaps the global render flag list, builds a separate `explore_tree`, then restores flags. This is an elegant reuse of the visibility computation for dual purposes.

**View-Local Effect State**: Camera effects (explosions, teleport folds) are stored directly in `view_data` as `render_effect`, `render_effect_phase`, etc. This tightly couples view parameters to their visual treatment but avoids a separate effects manager.

**Pre-allocated Buffer Pooling**: Subsystems like RenderVisTree pre-allocate to map-worst-case (`MAXIMUM_POLYGONS_PER_MAP`, `MAXIMUM_ENDPOINTS_PER_MAP`) at initialization. This avoids per-frame allocation overhead but means memory usage is determined by the largest possible map, not typical map sizes. This pattern is typical of 1990sΓÇô2000s game engines.

**Deterministic Tick-Driven Animation**: Transfer mode phases are computed from tick counts (exposed via `transfer_phase` parameter), not wall-clock time. This ensures networked multiplayer stays synchronized without frame-rate dependency. Phase wrapping uses fixed-point arithmetic (`FIXED_ONE` wraparound) for predictability.

## Data Flow Through This File

**Initialization Flow:**
1. `allocate_render_memory()` ΓåÆ pre-allocates visibility tree, sorter, placer buffers; wires cross-pointers between subsystems
2. `initialize_view_data()` ΓåÆ calculates projection matrices (FOV, world-to-screen scaling, cone angles) from camera parameters

**Per-Frame Render Flow:**
1. `render_view()` entry point receives camera state + target framebuffer
2. `update_view_data()` ΓåÆ recalculates camera vectors, applies FOV transitions, applies active effects (shake, fold)
3. `RenderVisTree.build_render_tree()` ΓåÆ ray-casts from camera through 2D geometry to find visible polygons; accumulates clipping flags
4. `RenderSortPoly.sort_render_tree()` ΓåÆ depth-orders visible polygons, refines clip windows
5. `RenderPlaceObjs.build_render_object_list()` ΓåÆ projects game objects (sprites, monsters), culls occluded ones, merges into sorted render list
6. Backend selection ΓåÆ Choose rasterizer (Rasterizer_SW vs. Rasterizer_Shader depending on `OGL_IsActive()`)
7. `RasPtr->Begin()` + `RenPtr->render_tree()` ΓåÆ Rasterize all visible geometry, applying transfer modes and clipping
8. `render_viewer_sprite_layer()` ΓåÆ Overlay weapon sprites with their own transfer mode applications
9. `RasPtr->End()` ΓåÆ Finalize rasterization (flushes GPU, swaps buffers, etc.)
10. Optional: `render_overhead_map()` or `render_computer_interface()` ΓåÆ Conditional 2D overlays

**Effect Application Flow:**
- `update_render_effect()` ΓåÆ increments effect phase, applies fold scale or camera shake based on effect type
- `instantiate_polygon_transfer_mode()` ΓåÆ modulates texture coordinates (translate, wobble) or distorts polygon geometry
- `instantiate_rectangle_transfer_mode()` ΓåÆ modulates sprite visibility (invisibility ΓåÆ tinted), size (fold ΓåÆ shrink), or clipping

## Learning Notes

**Fixed-Point Trig Tables**: The reliance on `cosine_table[]`, `sine_table[]`, and `NORMALIZE_ANGLE()` reflects 1994ΓÇô2000 era optimization. Modern CPUs compute sin/cos efficiently, but lookup tables guarantee determinism and avoid floating-point divergence in networked play. The tradeoff (cache pollution vs. determinism) favored determinism.

**Pure Render-Per-Frame Design**: Unlike modern engines using temporal coherence, occlusion hierarchies, or visibility caching, this engine rebuilds the visibility tree and sorts polygons completely from scratch each frame. This is simpler, works correctly with dynamic geometry (moving platforms, changing lights), and was acceptable on 2000s hardware. It doesn't scale to massive worlds but is predictable.

**Aspect-Ratio and FOV Gymnastics**: `initialize_view_data()` handles non-square pixels, non-square screens, and OpenGL's perspective projection model (which differs from the software rasterizer's oblique projection). The 1.3x FOV correction factor for OpenGL is a hard-coded fudge factor, suggesting the abstraction between software and GPU rendering was never perfect and was maintained via empirical tuning.

**Degenerate Case Handling**: The logic checking if polygon vertices lie under the view origin (`find_line_crossed_leaving_polygon()` in shake logic, media boundary checks) suggests the engine was burned by edge casesΓÇöprobably from player exploits or mod-created pathological geometry in early versions.

**Weapon Layer Composition**: Rendering weapons *after* the main scene (simple "always on top" 2D compositing) rather than using depth ordering or 3D billboarding suggests the original engine didn't have depth-of-field or transparency ordering. Modern approaches would use depth peeling or forward+ rendering, but this is simpler and sufficient for first-person view.

**Lua/XML Era (circa 2000)**: The modularization into separate rasterizer classes and the cross-reference pattern (RenderVisTree ΓåÆ RenderSortPoly ΓåÆ RenderPlaceObjs ΓåÆ RenderRasterize) suggest post-2000 refactoring, likely as the Aleph One community extended the codebase with Lua scripting and MML (Marathon Markup Language) configuration. The original monolithic render.c was decomposed into smaller classes for maintainability.

## Potential Issues

**Backend Abstraction Leakage**: While the Rasterizer base class provides interface abstraction, `RenderRasterize_Shader.h/cpp` is GPU-specific with GLSL shader code, suggesting the abstraction breaks down. A truly agnostic abstraction would hide shader concepts from render.cpp, but instead render.cpp conditionally includes and instantiates GPU-specific classes. This is fragile if new backends are added.

**Global Static State Coupling**: The use of static instances (`static RenderVisTreeClass RenderVisTree`, etc.) creates implicit module coupling. Changing one subsystem's state affects all callers. Thread-safety is not obviousΓÇöif the render loop runs on a separate thread (common in 2000s engines), these statics could race with Lua or input processing.

**Transfer Mode Phase Wrapping**: The `transfer_phase` parameter wraps at `FIXED_ONE`, but the file doesn't explicitly document what happens if animation frequencies drift (e.g., different transfer modes running at different cadences in the same frame). The wobble/wander logic multiplies phase by different factors (`* 2` for fast modes, `* 15` for faster), which could cause visual aliasing if not carefully tuned.

**No Culling or LOD**: The visibility tree determines which polygons are visible, but there's no frustum culling, occlusion culling, or level-of-detail. Every visible polygon is projected and rasterized at full detail. For large, complex maps, this could be a bottleneck, though the engine's design (many small rooms connected by corridors) naturally limits visible geometry.

**Marathon 1 Exploration Hack**: The `explore_tree` and `explore_view` are static singletons that shadow the main render tree. If M1 exploration runs on a separate thread or is invoked recursively, buffer corruption could occur (e.g., if the main render is interrupted mid-tree-build). The render flag list swap is an attempt to isolate, but it's not thread-safe.
