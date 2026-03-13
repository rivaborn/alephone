# Source_Files/RenderMain/render.h - Enhanced Analysis

## Architectural Role

This header is the **dispatcher facade** for Aleph One's rendering pipelineΓÇöthe thin coordination layer between game simulation (GameWorld) and specialized rendering subsystems (RenderVisTree, RenderSortPoly, RenderPlaceObjs, RenderRasterize). It exports the view state container (`view_data`) and entry points (`render_view`, `start_render_effect`) that game loop calls once per frame. Critically, **render.h defines no rendering logic itself**; it orchestrates calls to highly specialized modules that handle visibility culling, polygon sorting, object placement, and rasterization across three distinct backends (software, OpenGL classic, shader-based GPU).

## Key Cross-References

### Incoming (who depends on this)
- **GameWorld** (`marathon2.cpp::update_world`): Calls `render_view()` every frame with current player view; passes camera state via initialized `view_data`
- **Game Loop** (`MainController`): Calls `allocate_render_memory()` during startup, `render_overhead_map()` and `render_computer_interface()` in post-frame UI pass
- **ViewControl subsystem**: Populates view_data FOV fields via accessor macros (`View_FOV_Normal()`, `View_FOV_TunnelVision()`)
- **Input/Player** subsystems: Update `view_data.yaw`, `view_data.pitch`, `view_data.origin` based on player movement and aiming

### Outgoing (what this file depends on)
- **RenderVisTree**: `build_render_tree()` determines polygon visibility via ray casting (called from `render_view()`)
- **RenderSortPoly**: Depth-orders visible polygons using `half_cone` and clipping bounds from `view_data`
- **RenderPlaceObjs**: Projects sprites/objects using `view_data` screen-to-world transforms
- **RenderRasterize** / **Rasterizer backends**: Clip and rasterize polygons using `view_data` projection parameters
- **AnimatedTextures**: Uses `tick_count` to animate wall textures frame-by-frame
- **OGL_Render / OGL_Textures / Rasterizer_Shader**: OpenGL rendering backends query `view_data` for camera and effect state
- **RenderFlagList** global: Shared per-frame working buffer marked/queried during visibility traversal and rasterization

## Design Patterns & Rationale

| Pattern | Evidence | Rationale |
|---------|----------|-----------|
| **Parameter Object** | `view_data` clusters 20+ loosely related fields (camera, projection, effects, UI state) | Simplifies function signatures across dispatch chain; consolidates frame state in one mutable struct |
| **Visitor/Traversal** | `render_flags[]` array marks geometry (polygons, endpoints, sides, lines) visited during visibility tree construction | Reuses bitfield buffer across multiple traversal phases without temporary allocations |
| **Backend Abstraction** | Rasterizer interface hidden; software/OpenGL/Shader backends swapped at runtime | Late-1990s pattern before GPU became dominant; software rasterizer kept as fallback |
| **Tick-driven Animation** | `tick_count` field drives all effects and texture animation deterministically | Matches 30 FPS fixed-tick simulation (GameWorld); enables replay recording and client-side prediction without floating-point drift |
| **Effect State Machine** | `effect` + `effect_phase` pair; phase incremented each frame until effect completes | Simple single-effect queue; no effect blending/composition (pre-GPU limitation) |
| **Lazy FOV Interpolation** | `field_of_view` (current) vs. `target_field_of_view` (goal); bridge over multiple frames | Smooth FOV transitions for tunnel vision / sniper scope without abrupt jump |

## Data Flow Through This File

**Entry**: `render_view(view_data*, bitmap)` called from main game loop
1. `view_data` pre-populated by GameWorld with player position, orientation, animation tick
2. **Visibility phase**: `build_render_tree()` uses `origin`, `yaw`, `pitch`, `half_cone` to ray-cast and mark visible polygons in `render_flags[]`
3. **Projection phase**: `world_to_screen_x`, `world_to_screen_y`, `horizontal_scale`, `vertical_scale` cached; used to transform worldΓåÆscreen coordinates
4. **Effect phase**: If `effect Γëá 0`, `effect_phase` increments; modulates shading tables, viewport, or screen distortion
5. **Rasterization**: Visible polygons sorted, clipped, rasterized via active backend; transfer modes applied per-polygon via `instantiate_polygon_transfer_mode()` helper
6. **UI overlay**: `overhead_map_active` and `terminal_mode_active` flags trigger `render_overhead_map()` and `render_computer_interface()` post-composition

**State persistence**: `tick_count` global increments; drives animated textures, particle effects, sound updates across frame boundary.

## Learning Notes

- **Era signature**: This architecture mirrors late-1990s raycaster + polygon engines (e.g., Quake, Unreal). Camera math is entirely CPU-side; FOV/projection matrices computed per-frame, not GPU-resident.
- **Bolt-on features** (LP comments): `show_weapons_in_hand`, `tunnel_vision_active`, `landscape_yaw`, `mimic_sw_perspective`, `billboard_xy` are all post-hoc additions, not core to original designΓÇöevidenced by comments dating to FebΓÇôNov 2000.
- **Double-buffering pattern**: `interpolated_world.h` (architecture overview) suggests world snapshots exist; render_view samples interpolated state for smooth >30 FPS motion while maintaining deterministic 30 FPS simulation tick.
- **Transfer modes** abstracted: `instantiate_rectangle_transfer_mode / instantiate_polygon_transfer_mode` delegate tinting, static, wobble, landscape effects to per-object callouts rather than monolithic rasterizer loopΓÇöearly form of plugin-style rendering.

## Potential Issues

- **Monolithic parameter object**: `view_data` mixes camera state, projection caches, effect animation, UI flags, and media queriesΓÇöviolates single responsibility. A frame-local context object would better separate concerns.
- **Single-effect queue**: Only one effect (fold, explosion) active at a time; no effect blending or stacking. Modern engines layer post-process effects.
- **Static render flag buffer**: `RenderFlagList` sized once at startup based on `MAXIMUM_POLYGONS_PER_MAP`; can waste memory on small maps or fail on huge custom maps. Dynamic resizing would be safer.
- **Implicit tick sync**: `tick_count` increments in `render_view()`; coupling to render frequency breaks if someone decouples simulation from render (e.g., variable timestep, physics substepping). Explicit tick-passing would be clearer.
