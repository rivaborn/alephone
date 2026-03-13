# Source_Files/GameWorld/flood_map.h

## File Purpose
Header file declaring the pathfinding and flood-fill system for the game engine. Provides interfaces for computing paths between world polygons, moving entities along paths, and performing flood-fill operations with customizable cost algorithms for AI navigation and reachability analysis.

## Core Responsibilities
- Declare pathfinding memory management and path lifecycle functions
- Define flood-fill algorithm modes and customizable cost calculation interface
- Provide path creation, traversal, and deletion operations
- Declare flood-map computation with multiple algorithm strategies
- Support random sampling from computed flood areas with optional spatial bias
- Enable cost-based path optimization via user-supplied cost functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `cost_proc_ptr` | typedef (function pointer) | Callback for per-edge pathfinding cost; returns int32 cost given source polygon, edge, destination polygon, and opaque caller data |
| Flood modes enum | enum | Four pathfinding strategies: `_depth_first` (unsupported), `_breadth_first` (fast for large domains), `_flagged_breadth_first` (stores 4-byte flags in caller_data), `_best_first` |

## Global / File-Static State
None.

## Key Functions / Methods

### new_path
- **Signature:** `short new_path(world_point2d *source_point, short source_polygon_index, world_point2d *destination_point, short destination_polygon_index, world_distance minimum_separation, cost_proc_ptr cost, void *data)`
- **Purpose:** Compute a path between two points in different polygons with minimum separation constraint.
- **Inputs:** Source/destination world points and polygon indices; minimum separation distance; optional cost function pointer and opaque caller data.
- **Outputs/Return:** Short path ID for use with `move_along_path` and `delete_path`.
- **Side effects:** Allocates and populates internal path data structure; invokes cost function during pathfinding if non-NULL.
- **Calls:** User-supplied cost function (if provided).
- **Notes:** If `cost_proc` is NULL, defaults to polygon area as cost metric. Cost function receives source polygon, intermediate line index, destination polygon.

### move_along_path
- **Signature:** `bool move_along_path(short path_index, world_point2d *p)`
- **Purpose:** Advance along a previously computed path, updating the provided point.
- **Inputs:** Path ID; pointer to world point to update.
- **Outputs/Return:** Boolean; true if point updated and path continues, false if path end reached.
- **Side effects:** Modifies input point; advances internal path traversal state.
- **Calls:** None visible.
- **Notes:** Incremental per-frame usage; call repeatedly to follow a path to completion.

### flood_map
- **Signature:** `short flood_map(short first_polygon_index, int32 maximum_cost, cost_proc_ptr cost_proc, short flood_mode, void *caller_data)`
- **Purpose:** Compute reachable polygons from a starting polygon within a cost budget using specified algorithm.
- **Inputs:** Starting polygon index; maximum cost limit; cost function (NULL = polygon area); flood algorithm mode; opaque caller data.
- **Outputs/Return:** Short result (likely polygon count or identifier).
- **Side effects:** Builds flood-map structure in global state; invokes cost function for edge traversal.
- **Calls:** User-supplied cost function.
- **Notes:** Breadth-first noted as significantly faster for large domains than best-first. Flagged breadth-first mode interprets `caller_data` as `int32*` pointing to 4 bytes of flags per polygon.

### choose_random_flood_node
- **Signature:** `void choose_random_flood_node(world_vector2d *bias)`
- **Purpose:** Select a random polygon from the most recent flood-map result with optional spatial bias.
- **Inputs:** Optional pointer to 2D bias vector (NULL for unbiased selection).
- **Outputs/Return:** None (result stored in global flood state).
- **Side effects:** Modifies global flood state.
- **Calls:** None visible.

### Trivial helpers
- `allocate_pathfinding_memory`, `reset_paths`, `delete_path`, `allocate_flood_map_memory`, `reverse_flood_map`, `flood_depth` ΓÇö lifecycle and state query operations.

## Control Flow Notes
Likely used in AI and entity systems. Initialization phase calls `allocate_pathfinding_memory()` and `allocate_flood_map_memory()` once. Per-request: `new_path()` for entity-to-target paths or `flood_map()` for area reachability. Per-frame: `move_along_path()` updates entity positions along active paths. Cleanup: `delete_path()` when path completes or is abandoned.

## External Dependencies
- **Includes:** `world.h` ΓÇö provides `world_point2d`, `world_distance`, `world_vector2d` types, polygon indices, and world coordinate systems.
- **Defined elsewhere:** Memory allocators, path and flood-map storage, cost calculations, random selection logic.
