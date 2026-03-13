# Source_Files/GameWorld/flood_map.cpp - Enhanced Analysis

## Architectural Role

`flood_map.cpp` is the **core polygon-based graph search engine** powering monster AI pathfinding and waypoint generation in Aleph One. It is called iteratively by the game loop to incrementally explore the polygon graph, allowing AI decisions to be distributed across frames rather than blocking on a single pathfinding call. The stateful design (persistent search tree in static globals) reflects 1990s architecture constraintsΓÇöbefore coroutines/generatorsΓÇöand enables the caller (monsters.cpp, pathfinding.cpp) to interleave pathfinding with other simulation work.

## Key Cross-References

### Incoming (who depends on this)
- **monsters.cpp**: Monster AI loop calls `flood_map()` to build target paths, then `reverse_flood_map()` to walk them
- **pathfinding.cpp** (`allocate_pathfinding_memory()`, path traversal): Uses flood search results to generate waypoints and guide monster movement
- **devices.cpp / map.cpp**: Indirect dependents; platform/trigger logic may trigger AI repathing
- Comment mentions **fareast.c** implements `_depth_first` mode (unsupported stub here)

### Outgoing (what this file depends on)
- **map.cpp**: `get_polygon_data()` (read polygon adjacency), `changed_polygon()` (invalidate pathfinding on geometry changes)
- **world.cpp**: `find_center_of_polygon()` (for bias calculation in `choose_random_flood_node()`), `global_random()` (RNG for path randomization)
- **CSeries**: `objlist_set()` macro (array fill), assertion/logging
- **Dynamic world state**: `dynamic_world->polygon_count`, polygon adjacency lists (read-only)

## Design Patterns & Rationale

**Stateful Incremental Search**: Rather than blocking on a complete pathfinding call, the search tree persists across multiple `flood_map()` calls via static globals (`nodes`, `visited_polygons`, `last_node_index_expanded`). This allows AI pathfinding to be distributed across game ticksΓÇömonster moves one step while pathfinding advances one node. This pattern predates modern async/await and generator-based solutions.

**Multiple Search Modes**: 
- `_best_first`: Explores lowest-cost nodes first ΓåÆ optimal paths but slower for large maps
- `_breadth_first`: Explores nodes in insertion order ΓåÆ faster but explores more polygons
- `_flagged_breadth_first`: Breadth-first with caller-provided flags (used for state propagation during search, e.g., tracking visited areas)
- `_depth_first`: Unsupported (caller-implemented); suggests historical DFS mode that was never ported

**Cost Function Indirection** (`cost_proc`): Allows pathfinding callers to define traversal cost (e.g., prefer open areas, penalize lava) without modifying flood_map. If NULL, defaults to polygon area.

**Parent Pointer Backtracking**: Enables O(depth) path reconstruction via `reverse_flood_map()` without storing full paths; trades space for CPU.

## Data Flow Through This File

```
Caller (AI system)
  Γåô
flood_map(START, max_cost, cost_fn, mode, caller_data)
  Γö£ΓöÇ [INIT] Clear visited_polygons, seed nodes list with START
  ΓööΓöÇ [PER CALL] 
      Γö£ΓöÇ Select next node (best-first, breadth-first, or custom)
      Γö£ΓöÇ Mark as expanded
      Γö£ΓöÇ Enqueue adjacent polygons via add_node()
      ΓööΓöÇ Return expanded polygon index
  Γåô
[Caller repeats until returns NONE or destination reached]
  Γåô
reverse_flood_map() loop
  Γö£ΓöÇ Walk backward via parent_node_index
  ΓööΓöÇ Return polygons in path (source ΓåÆ dest)
```

**Key state transitions**: `UNVISITED` ΓåÆ enqueued (in nodes list) ΓåÆ expanded (bit 0x8000) ΓåÆ backtracked.

## Learning Notes

**1990s Algorithm Heritage**: 
- Fixed-size static arrays (255 nodes max); modern engines use dynamic queues
- Manual flag bit-packing (`NODE_IS_EXPANDED` macro) to save 2 bytes per node
- No STL containers; pure C struct array management
- Deterministic RNG via `global_random()` for replay consistency (important for networked games)

**Search Mode Evolution**: The presence of unimplemented `_depth_first` and historical comments ("Monday, September 5, 1994...") reveal this code predates optimization passes. Best-first and breadth-first represent a pragmatic tradeoff between optimality and speed.

**Biased Random Selection**: `choose_random_flood_node()` uses dot product to prefer nodes in a given directionΓÇösimple but effective for steering monster movement toward objectives without explicit A* heuristics.

## Potential Issues

1. **Infinite Loop in `choose_random_flood_node()`**: Comment explicitly warns: *"on very small maps, choose_random_flood_node() may not terminate"* if all expanded nodes are unsuitable. Retry limit (`MAXIMUM_BIASED_RETRIES = 10`) provides escape, but if exhausted, loop continues checking bias indefinitely.

2. **Silent Node Pool Overflow**: `add_node()` silently fails if `node_count >= MAXIMUM_FLOOD_NODES` (255). No error signaling; caller receives `NONE` from `flood_map()` and assumes search exhausted, masking the true cause (pool full, not frontier empty).

3. **Global State Not Thread-Safe**: Static globals assume single-threaded game loop. If pathfinding and monster AI ran in parallel threads, race conditions on `nodes`, `visited_polygons`, `last_node_index_expanded` would corrupt search state. Aleph One is single-threaded, so not an issue in practice.

4. **Cost Function Re-entrancy**: If `cost_proc` calls back into pathfinding (e.g., recursive AI decision), global state gets clobbered. Not inferable from code, but worth noting as a fragile assumption.

5. **No Cost Monotonicity Check**: Best-first assumes costs are monotonically increasing along paths (or at least consistent). If `cost_proc` returns inconsistent values, optimality is not guaranteed. Modern Dijkstra implementations validate this; flood_map trusts the caller.
