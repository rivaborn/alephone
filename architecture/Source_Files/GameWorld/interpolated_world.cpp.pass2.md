# Source_Files/GameWorld/interpolated_world.cpp - Enhanced Analysis

## Architectural Role

This file implements the **latency-hiding interpolation layer** that decouples the fixed 30 FPS game simulation from variable render framerates (60+ FPS). It's a critical performance enabler for the Aleph One engine: players perceive smooth motion while the deterministic simulation remains unaffected. The file acts as a **double-buffered snapshot manager**, capturing world state at each tick boundary and blending between ticks per-frame based on elapsed wall-clock time. This is foundational to the rendering pipeline (RenderMain) and sits at the boundary between deterministic game logic (GameWorld) and frame-rate-variable display (Rendering/Screen).

## Key Cross-References

### Incoming (who depends on this file)

- **render.cpp/RenderRasterize_Shader.cpp**: Call `update_interpolated_world(heartbeat_fraction)` and `interpolate_world_view(heartbeat_fraction)` every render frame with the fractional progress [0, 1] between ticks
- **render.cpp**: Calls `get_heartbeat_fraction()` to calculate interpolation parameter based on elapsed machine ticks; also checks `world_is_interpolated` flag to decide whether to interpolate or use raw world state
- **screen.cpp/interface.cpp**: May call init/enter/exit lifecycle functions during mode transitions (fullscreen, pause, replay)
- **marathon2.cpp**: Calls `enter_interpolated_world()` at each 30 FPS tick boundary to snapshot fresh world state
- **weapons.cpp**: Via `track_contrail_interpolation()` to register projectile positions for particle effect interpolation
- **HUD/UI rendering**: Implicitly depends on interpolated camera state via `world_view` global modifications

### Outgoing (what this file depends on)

- **map.h/cpp**: Calls `find_new_object_polygon()`, `get_line_data()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()` to maintain spatial object lists during polygon boundary crossings; reads `map_polygons`, `map_sides`, `map_lines` global arrays
- **render.h/cpp**: Calls `update_world_view_camera()` to capture camera state at tick boundary; accesses `world_view` global
- **ephemera.h/cpp**: Calls `get_ephemera_data()`, `remove_ephemera_from_polygon()`, `add_ephemera_to_polygon()` for particle list management during interpolation
- **weapons.h**: Reads `weapon_display_information` structure; calls `get_weapon_display_information()` iterator; interpolates weapon display data
- **dynamic_limits.h**: Queries `get_dynamic_limit(_dynamic_limit_ephemera)` for pool sizing
- **preferences.h**: Calls `get_fps_target()` to check if interpolation should be active
- **Movie.h**: Checks `Movie::instance()->IsRecording()` to adjust heartbeat calculation for movie export
- **player.h**: Reads global `objects` array (player and projectiles)
- **Timing**: Uses `machine_tick_count()` for elapsed time calculation; reads `TICKS_PER_SECOND`, `MACHINE_TICKS_PER_SECOND` constants

## Design Patterns & Rationale

### Double-Buffering Pattern
The previous/current tick snapshot vectors (`previous_tick_objects`, `current_tick_objects`, etc.) implement **temporal ping-pong buffering**. This enables lock-free, frame-independent interpolation: rendering reads from both buffers simultaneously without blocking the 30 FPS game loop. Rationale: avoids synchronization overhead while preserving determinism of original simulation.

### Per-Frame Interpolation Parameter
The `heartbeat_fraction` mechanism **abstracts away wall-clock timing**. Callers don't need to know elapsed machine ticks; they receive a normalized [0, 1] parameter where 0 = previous tick snapshot, 1 = current tick snapshot. This decoupling enables:
- Special handling during movie export (use export phase instead of machine ticks)
- Replay slow-motion (scale heartbeat by replay speed multiplier)
- Movie recording (snap heartbeat to frame boundaries)

### Speed-Based Interpolation Rejection
The `should_interpolate()` and `get_object_speed_limit()` functions implement **adaptive interpolation rejection**. Fast-moving objects (projectiles) have a higher speed threshold (~2.0 world units) vs. normal objects (~0.5 world units). Rationale: prevents temporal aliasing artifacts where fast objects would appear to teleport or blur. Trade-off: fast objects occasionally jerk instead of moving smoothly, but this is less noticeable than interpolation artifacts.

### Polygon Boundary Crossing During Interpolation
When an object moves between polygons during a frame, the code calls `find_new_object_polygon()` and relinks the object into the new polygon's object list via `remove_object_from_polygon_object_list()` ΓåÆ `add_object_to_polygon_object_list()`. Rationale: maintains spatial invariants required by collision detection, visibility determination, and media (liquid) containment checks. This ensures render-time queries (e.g., "what objects are in this polygon?") remain correct despite interpolation.

### Contrail Tracking System
The `ContrailInfo` structure and `track_contrail_interpolation()` function solve a specific problem: **particle effects (contrails, smoke trails) that spawn and don't move** need the projectile's *previous-tick* position for smooth visual streaking. Storing the projectile location separately allows contrail rendering to interpolate from previous tick position (not moved) to current tick position (moved), creating the visual effect of a trail. Rationale: projectiles move too fast for normal interpolation; this preserves visual continuity of transient effects.

### Angle Wraparound Handling
The `lerp_angle()` and `lerp_fixed_angle()` functions handle the circular domain problem: angles wrap at 360┬░. Naive lerp of (1┬░ to 359┬░) would produce a 179┬░ swing through 180┬░. These functions **choose the shortest angular path**, adding FULL_CIRCLE to one endpoint if the direct path is > 180┬░. Rationale: camera and player rotation should never jerk across the discontinuity during interpolation. The fixed-angle variant uses `FIXED_ONE` constants to preserve precision in fixed-point arithmetic (likely used internally for weapon pitch/yaw).

## Data Flow Through This File

### Capture Phase (per 30 FPS tick)
1. **enter_interpolated_world()** is called at tick boundary
2. Snapshots current world state: objects (position, polygon, flags), polygons (floor/ceiling heights), lines (adjacent heights), sides (texture offsets), ephemera, camera, weapons
3. Stores in `current_tick_*` vectors; swaps previous ΓåÉ current
4. Records `start_machine_tick` for heartbeat calculation
5. Game logic then modifies live world state for next tick

### Interpolation Phase (per render frame, up to 60+ times per game tick)
1. Render loop calls `get_heartbeat_fraction()` ΓåÆ calculates elapsed time since `start_machine_tick` as fraction [0, 1]
2. Calls `update_interpolated_world(fraction)` ΓåÆ iterates visible polygons/objects/ephemera, linearly interpolates position/height/angles, handles polygon crossings
3. Calls `interpolate_world_view(fraction)` ΓåÆ interpolates camera position/rotation, handles camera polygon crossing, updates `world_view->origin/yaw/pitch/etc.`
4. Calls `interpolate_weapon_display_information()` ΓåÆ interpolates HUD weapon positions
5. Renderer consumes modified `objects`, `map_polygons`, `world_view` state with interpolated values
6. At next tick boundary, cycle repeats: current state becomes previous, new snapshot captured

### Cleanup Phase (on exit/level transition)
1. **exit_interpolated_world()** copies `current_tick_*` snapshots back into live world state
2. Restores engine to clean state before interpolation buffers are deallocated

## Learning Notes

### Historical Context: 1990s-Era Smoothing Technique
This file shows how **Marathon (1994) engine designers solved the 30 FPS ΓåÆ 60+ FPS problem** before GPU-assisted framebuffer interpolation became standard. The approach is pure software simulation state blendingΓÇöa foundational technique still used in deterministic networked games (fighting games, some esports titles) where replay integrity is critical.

### Precision Trade-offs
- **int16_t positions + linear lerp**: 16-bit positions are limited to ~32K world units; lerp is basic floating-point blending. Compare to modern engines using 32-bit floats with Catmull-Rom splines.
- **Fixed-point angles (FIXED_ONE)**: Internal camera angles use fixed-point to avoid floating-point precision drift during accumulation. Shows attention to determinism.
- **Speed limits are 2D only**: Ignores vertical velocity (Z delta), so a projectile could move fast vertically and still interpolate. Likely a simplification based on game physics (most projectiles are fast horizontally, slow vertically).

### No Vector Interpolation
Polygon heights, line properties, and texture coordinates are interpolated individually (floor_height, ceiling_height, y0 offset). No higher-order spline interpolation is used. This suggests the original designers prioritized simplicity and predictability over smooth curves.

### Ephemera (Particles) Are Snapshotted But Not Fully Interpolated
The code stores `previous_tick_ephemera` and `current_tick_ephemera` but the interpolation loop doesn't interpolate individual ephemera positionsΓÇöit only relinks them to new polygons if they cross boundaries. This is likely because particles are often short-lived and already have their own animation/lifetime, so lerping their position would be redundant.

### Movie Integration Points
Calls to `Movie::instance()->IsRecording()` in `get_heartbeat_fraction()` show deep integration with the replay system. Suggests that replay validation depends on heartbeat being deterministic and reproducible.

## Potential Issues

### Stale Contrail References
The `contrail_tracking[effect_index]` array stores an `int16_t projectile_index` to link effects to projectiles. If the projectile object slot is reused (object dies, new entity allocated in same slot), the contrail tracking becomes stale and may track the wrong object. Mitigation: likely handled by clearing `contrail_tracking[i].projectile_index = NONE` when an object is destroyed.

### Polygon Crossing at Frame Boundaries
If an object straddles a polygon boundary during interpolation, `find_new_object_polygon()` is called. If this function is unreliable or slow (flood-fill search), objects could briefly appear in wrong polygons, causing:
- Visibility clipping artifacts (objects hidden when they shouldn't be)
- Collision detection failures (player can pass through geometry briefly)
- Media containment errors (player appears to exit liquid without taking damage)

### Camera Polygon Crossing Visibility Glitches
The camera can cross polygon boundaries during interpolation via `interpolate_world_view()`. If the interpolated camera location doesn't match the actual camera polygon, the visibility tree (built in RenderVisTree.cpp) could show incorrect geometry. This is partially mitigated by calling `find_new_object_polygon()` on the camera, but if the interpolated location is in a transition zone, visibility might flicker.

### Speed Limit Threshold Tuning
The hardcoded `default_speed_limit = WORLD_ONE_HALF` and `projectile_speed_limit = WORLD_ONE` are magic constants. If map geometry is rescaled or physics constants change, these thresholds may become incorrect:
- Too low: normal objects jerk unpredictably
- Too high: fast objects show temporal aliasing (ghosting, streaking)

### Weapon Display Interpolation Search
The `interpolate_weapon_display_information()` function searches a circular buffer to match weapon display data. If the search fails (stale data, wrong index), weapon HUD could briefly display wrong positions or skip frames.

---

## Summary

**interpolated_world.cpp** is a bridge between the deterministic 30 FPS simulation and smooth high-fps rendering. Its key insight is **temporal double-buffering with adaptive interpolation rejection**ΓÇöa pragmatic 1990s solution that trades occasional jerkiness for rock-solid frame rate and determinism. The file shows sophisticated handling of circular domains (angles), spatial invariants (polygon lists), and time decoupling (heartbeat fraction). It remains relevant in modern engines where replay/netcode determinism matters more than pure visual smoothness.
