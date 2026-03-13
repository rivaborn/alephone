# Source_Files/GameWorld/pathfinding.cpp - Enhanced Analysis

## Architectural Role

This file is the **waypoint generator** for the monster/NPC AI subsystem, bridging the **flood-fill graph search** (`flood_map.h`) and **map geometry layer** (`map.h`). It sits at a critical juncture: monsters request paths via `new_path()`, the pathfinding layer computes routes across the polygon graph, and the result is a reusable waypoint sequence that guides entity movement frame-by-frame. The global pool design reflects early 1990s real-time constraintsΓÇöpre-allocated, bounded, reused per tick to avoid dynamic allocation overhead.

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld/monsters.cpp** ΓÇö Monster AI behavior drives `new_path()` calls when acquiring/losing targets; calls `move_along_path()` each frame during pursuit; calls `delete_path()` on death or goal completion  
- **GameWorld/devices.cpp, platforms.cpp** ΓÇö May request paths for interactive entities (teleporters, moving platforms tracking targets)  
- **RenderOther/overhead_map.cpp** ΓÇö Calls `path_peek()` to visualize active paths for debugging  
- **Misc/dynamic_limits.cpp** ΓÇö Triggers `allocate_pathfinding_memory()` when `MAXIMUM_PATHS` resource limit changes at runtime  

### Outgoing (what this file depends on)
- **GameWorld/flood_map.h** ΓÇö `flood_map()`, `reverse_flood_map()`, `flood_depth()`, `choose_random_flood_node()` ΓÇö the core graph search and path extraction  
- **GameWorld/map.h** ΓÇö `find_shared_line()`, `get_line_data()`, `get_endpoint_data()` ΓÇö geometry queries for waypoint placement at polygon boundaries  
- **Misc/dynamic_limits.h** ΓÇö `get_dynamic_limit(_dynamic_limit_paths)` ΓÇö runtime-tunable path pool size  

## Design Patterns & Rationale

1. **Object Pool (Free List)**: The `paths` array is pre-allocated once and recycled. `step_count = NONE` signals "free slot." This avoids malloc/free thrashing in a 30 FPS real-time loop; pools are bounded to prevent denial-of-service (monsters can't spam path requests).

2. **Two-Phase Path Extraction**:
   - **Phase 1** (`flood_map`): Forward search from source ΓåÆ destination polygon (breadth-first)  
   - **Phase 2** (`reverse_flood_map`): Backtrack the flood-fill tree to extract waypoints  
   This separation allows the flood-fill layer to remain stateless about path geometry; pathfinding.cpp handles the traversal details.

3. **Cost Function Delegation** (`cost_proc_ptr cost`): Callers supply a custom cost metric (e.g., "avoid lava," "prefer flat terrain"). This allows monster AI to express preferences without coupling pathfinding logic to AI heuristics.

4. **Graceful Degradation for Unreachable Targets**:
   - If destination is reachable: `step_count = depth + 1` (optimal path)  
   - If unreachable: `step_count = depth` + random node selection (biased random walk in feasible region)  
   This design prevents pathfinding thrashing: unreachable monsters get a reasonable attempt rather than constant re-planning.

5. **Elevation Clearance Handling** (`minimum_separation`): Endpoints marked `ENDPOINT_IS_ELEVATION` (stairs, ramps) subtract from the traversable range when computing waypoints. This prevents monsters from clipping through narrow ledges.

## Data Flow Through This File

```
Input: new_path(source_polygon, dest_polygon, minimum_separation, cost_fn)
  Γåô
flood_map() ΓåÆ breadth-first search of polygon graph (delegates to flood_map.h)
  Γåô
reverse_flood_map() ΓåÆ trace parent pointers back to source (loop until NONE)
  Γåô
For each polygon transition: 
  calculate_midpoint_of_shared_line() 
    ΓåÆ find shared boundary line between adjacent polygons
    ΓåÆ apply minimum_separation offset from elevation endpoints
    ΓåÆ interpolate waypoint along line using global_random()
  Γåô
Path structure: [current_step=0, step_count=N, points[0..N-1]]
  Γåô
Output: path_index (or NONE on failure)

Usage Loop:
  move_along_path(path_index) ΓåÆ *p = points[current_step++]
    Γåô (repeat per frame until current_step == step_count)
  delete_path(path_index) ΓåÆ mark NONE, free for reuse
```

**State Transitions**:  
- `step_count = NONE` (free) ΓåÆ allocated by `new_path()` ΓåÆ active during traversal ΓåÆ marked `NONE` by `delete_path()` or `move_along_path()` at completion

## Design Rationale & Tradeoffs

The choice of **breadth-first flood-fill on a polygon graph** reflects the map representation: Marathon uses convex polygons as the navigation domain. Modern engines might use A* with heuristics or pre-computed visibility meshes, but here the simplicity suits:
- Map sizes are bounded (not open-world)
- Flood-fill is cache-friendly and deterministic (good for network replays)
- Pre-computed costs mean callers can weight terrain preference at request time

The **waypoint placement at polygon boundaries** (rather than polygon centers) produces smoother paths through narrow gaps. However, comments reveal unfinished work: the algorithm doesn't "cut corners"ΓÇöif three polygons form a tight L-shape, the path traces the L rather than shortcutting.

## Learning Notes

1. **Early Game Engine Philosophy**: This code exemplifies 1990s game architectureΓÇöno dynamic memory allocation in the hot path, bounded pools, deterministic state. Modern engines have garbage collection and heap allocation but sacrifice predictability.

2. **Separation of Navigation & Pathfinding**:  
   - `flood_map.h` = graph search (algorithm-agnostic)  
   - `pathfinding.cpp` = waypoint extraction (geometry-aware)  
   This split allows flood-fill reuse in other subsystems (visibility, sound propagation).

3. **Cost Function Pointers as a Design Pattern**: Rather than hardcoding path heuristics, the caller injects cost metrics. This is a pre-C++ template era patternΓÇömodern code might use polymorphism or functors, but it's effective for tight coupling control.

4. **Elevation Handling Complexity**: The `minimum_separation` logic handles a geometry edge case (stairs clipping monsters). This suggests real maps had intricate polygon configurations that required careful waypoint placement.

5. **Debug Validation (`VERIFY_PATH_SYNC`)**: The validation buffer suggests determinism bugs in the pastΓÇöpaths diverging between runs. Likely a regression detector for platform-specific RNG or floating-point precision issues.

## Potential Issues

1. **Random Path Bias**: The random destination selection (`choose_random_flood_node()`) may be skewed toward central polygons in the flood-fill tree, producing non-uniform exploration. The code itself notes this: "random paths aren't really random."

2. **Minimum Separation Edge Case**: When `range <= 0` after elevation offsets, the code falls back to the line midpointΓÇöthis may place waypoints in unreachable positions if the line is blocked by geometry above/below it.

3. **No Corner Cutting**: Waypoints on polygon boundaries can produce paths that zig-zag through tight spaces rather than shortcutting. The TODO comments acknowledge this: "we should cut corners instead of blindly following map geometry."

4. **Shadow Polygon Problems**: Comments note "shadow polygons screw us up"ΓÇöinvalid geometry artifacts in the map can break pathfinding. No explicit detection/handling.

5. **Step Count Clipping**: Paths longer than `MAXIMUM_POINTS_PER_PATH` are silently truncated. Monsters reaching a clipped waypoint won't reach the true destination. No warning is logged.
