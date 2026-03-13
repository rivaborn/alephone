# Source_Files/GameWorld/flood_map.h - Enhanced Analysis

## Architectural Role

This header is the **AI navigation backbone** for monster pathfinding and reachability analysis in the GameWorld subsystem. It bridges the low-level polygon graph (map geometry from `world.h`) with high-level AI decision-making in `monsters.cpp` and entity placement in `items.cpp`. The flood-fill algorithm computes reachable polygon sets within a cost budget, enabling monsters to navigate dynamically changing geometry and enabling spawn placement logic to find valid areas without line-of-sight constraints.

## Key Cross-References

### Incoming (AI and Placement subsystems depend on this)
- **monsters.cpp**: Calls `flood_map()` to compute reachable destinations for pathfinding and target selection; uses `choose_random_flood_node()` to select biased random destinations for varied AI behavior
- **items.cpp**: Uses flood-fill results to identify valid polygon sets for item placement via `a1_swipe_nearby_items()` / `m2_swipe_nearby_items()`  
- **pathfinding.cpp**: Cooperates via `new_path()` to compute waypoint sequences between two points after flood-map reachability is established
- **marathon2.cpp** (main loop): Indirectly via monster activation and item spawning

### Outgoing (what this subsystem consumes)
- **world.h**: Provides `world_point2d`, `world_distance`, `world_vector2d` types and polygon indexing primitives
- **map.cpp**: Implicitly queries polygon adjacency and line data for edge traversal during flood-fill
- User-supplied `cost_proc_ptr` callbacks: Monster AI supplies custom cost functions to weight destination preference (e.g., avoid water)

## Design Patterns & Rationale

**Callback-based customization (`cost_proc_ptr`)**  
The function pointer pattern defers cost calculation to caller logic, avoiding tight coupling to specific heuristics. Default behavior (NULL pointer ΓåÆ polygon area) provides a sensible baseline; custom procs allow difficulty-scaled pathfinding (e.g., harder monsters ignore small areas) without recompiling. This reflects 1990s C idiom before C++ templates were standard.

**Multiple flood modes with explicit performance tradeoffs**  
- `_depth_first`: Marked unsupported; likely historical artifact
- `_breadth_first`: "significantly faster for large domains" per commentΓÇöhints at O(N) uniformity vs. priority queue overhead in `_best_first`
- `_flagged_breadth_first`: Interprets `caller_data` as `int32*` flag arrayΓÇöenables per-polygon state (visited, blocked, type mask) without reallocation
- `_best_first`: Priority-queue-based Dijkstra; slower but finds optimal-cost paths

This design reflects constraint of 1990s hardware: game ships with hardcoded difficulty-aware mode selection, not runtime heuristic tuning.

**Separation of path computation and traversal**  
`new_path()` ΓåÆ `move_along_path()` ΓåÆ `delete_path()` mirrors the incremental frame-by-frame game loop: compute once per decision, advance incrementally each frame. Avoids per-frame pathfinding recalculation overhead.

## Data Flow Through This File

1. **Input**: Monster AI or placement code calls `flood_map(start_polygon, budget, cost_fn, mode, caller_data)`
2. **Traversal**: Flood-fill explores adjacent polygons via edge/line queries (implicit via `world.h` polygon adjacency) until cost budget exhausted
3. **State accumulation**: Internal `add_node()` builds reachability set, marked by mode:
   - **Breadth-first**: Queue-based layer expansion
   - **Flagged**: Caller-provided flag buffer tracks per-polygon state
   - **Best-first**: Priority queue sorts by cumulative cost
4. **Output**: `flood_depth()` / `reverse_flood_map()` query results; `choose_random_flood_node(bias)` samples from reachable set (bias vector weights selection toward directions)
5. **Consumption**: Monster AI uses result to pick destination; pathfinding then computes waypoints from current position to destination

## Learning Notes

**Era-specific design choices:**
- **No STL containers** in the header; all state is opaque (allocated in .cpp via raw arrays)ΓÇöreflects pre-STL C++ codebase
- **Callback function pointers** instead of virtual methodsΓÇöavoids vtable overhead, easier to profile in debuggers on 1990s tools
- **Multiple modes instead of one algorithm**ΓÇöhints at iterative optimization: shipped _breadth_first as default, added _best_first later for players demanding optimal paths

**Idiomatic to early Aleph One:**
- "User data" pattern (opaque `void* caller_data`) recurs throughout (also in `cost_proc_ptr` signature) ΓÇö enables loose coupling but sacrifices type safety (no C++ generics era)
- The "default to sensible baseline" philosophy: NULL cost_proc defaults to polygon area, avoiding NULLs at call sites

## Potential Issues

- **Flagged breadth-first assumes 4-byte alignment** in caller-provided buffer; no bounds checking visible in headerΓÇöbuffer overrun risk if caller miscalculates size
- **No explicit thread safety guarantees** for concurrent flood-map queries (if netplay prediction threads call this)ΓÇölikely safe by convention (each player's monster list is thread-local), but undocumented
- **Random bias interpretation unclear**: `choose_random_flood_node(bias)` semantics (is bias normalized? does it guarantee bias direction preferred?) not documentedΓÇörisks AI unpredictability across difficulty levels
