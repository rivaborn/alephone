# Source_Files/GameWorld/pathfinding.cpp

## File Purpose
Implements waypoint pathfinding for monsters and NPCs. Creates sequences of 2D waypoints that guide entities across polygon-based map geometry using breadth-first flood-fill algorithms. Handles both destination-based paths and random biased paths for unreachable targets.

## Core Responsibilities
- Allocate and manage a global pool of reusable path structures
- Generate paths via flood-fill from source to destination polygon
- Calculate waypoint positions at shared polygon boundaries with minimum separation constraints
- Support random path generation when destinations are unreachable
- Track stepwise traversal along an active path
- Delete and reset paths for reallocation across game ticks
- Validate path consistency in debug builds

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `path_definition` | struct | A single path: holds current step index, total step count, and array of up to 63 waypoint coordinates |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `paths` | `path_definition*` | static | Dynamic array of all paths, indexed by path_index. Allocated at startup and on dynamic limit changes. |
| `path_validation_area` | `byte*` | static (debug) | Buffer for recording/validating path waypoints across runs (ifdef VERIFY_PATH_SYNC) |
| `path_validation_area_index` | `int32` | static (debug) | Write cursor in validation buffer |
| `path_run_count` | `short` | static (debug) | Counter tracking which pathfinding pass this is |

## Key Functions / Methods

### allocate_pathfinding_memory
- **Signature:** `void allocate_pathfinding_memory(void)`
- **Purpose:** Allocate or reallocate the global paths array and debug validation buffer. Reentrant to support dynamic limit changes at runtime.
- **Inputs:** None (reads `MAXIMUM_PATHS` from dynamic limits)
- **Outputs/Return:** None
- **Side effects:** Frees and reallocates `paths` array. If `VERIFY_PATH_SYNC` defined, frees and reallocates `path_validation_area`.
- **Calls:** `get_dynamic_limit()` (via macro), `new`, `delete`
- **Notes:** Uses C++ `new`/`delete` operators.

### reset_paths
- **Signature:** `void reset_paths(void)`
- **Purpose:** Mark all paths as unused at the start of each game tick; reset debug counters.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `step_count = NONE` for all paths. Increments `path_run_count` and resets `path_validation_area_index`.
- **Calls:** None
- **Notes:** Paths marked `NONE` become available for allocation by `new_path()`.

### new_path
- **Signature:** `short new_path(world_point2d *source_point, short source_polygon_index, world_point2d *destination_point, short destination_polygon_index, world_distance minimum_separation, cost_proc_ptr cost, void *data)`
- **Purpose:** Generate a path from source to destination (or random if destination is `NONE`). Returns path index or `NONE` on failure.
- **Inputs:**
  - `source_polygon_index` ΓÇô Starting polygon
  - `destination_point` ΓÇô Target location (or bias vector for random paths)
  - `destination_polygon_index` ΓÇô Target polygon (`NONE` for random path)
  - `minimum_separation` ΓÇô Clearance from polygon endpoints (handles elevations)
  - `cost`, `data` ΓÇô Cost function and user data for flood-fill
- **Outputs/Return:** Path index ΓëÑ 0 on success, or `NONE` if no free paths
- **Side effects:** Claims a free path slot; performs flood-fill; populates waypoints; if `VERIFY_PATH_SYNC`, records path data for validation.
- **Calls:** `flood_map()`, `reverse_flood_map()`, `flood_depth()`, `choose_random_flood_node()`, `calculate_midpoint_of_shared_line()`, debug helpers
- **Notes:** Destination reachable ΓåÆ `depth+1` waypoints; unreachable ΓåÆ `depth` waypoints with random destination. Waypoints clipped to `MAXIMUM_POINTS_PER_PATH`. Empty paths return `NONE`. Contains pre/post-generation asserts.

### move_along_path
- **Signature:** `bool move_along_path(short path_index, world_point2d *p)`
- **Purpose:** Retrieve the next waypoint and advance the path cursor.
- **Inputs:** `path_index` ΓÇô Path to traverse; `p` ΓÇô Output waypoint
- **Outputs/Return:** `true` if this was the last step; `false` otherwise
- **Side effects:** Increments `current_step`. Marks path `NONE` when exhausted.
- **Calls:** None
- **Notes:** Must follow `new_path()` success. Asserts path validity.

### delete_path
- **Signature:** `void delete_path(short path_index)`
- **Purpose:** Mark a path as unused.
- **Inputs:** `path_index` ΓÇô Path to free
- **Outputs/Return:** None
- **Side effects:** Sets path `step_count = NONE`
- **Calls:** None
- **Notes:** Asserts path is in use.

---

**Helper functions:**
- `calculate_midpoint_of_shared_line()` (private) ΓÇô Computes a waypoint along a shared polygon boundary, respecting minimum separation from endpoints.
- `path_peek()` (debug) ΓÇô Inspects a path's waypoint array for visualization.
- `GetNumberOfPaths()` (LP addition) ΓÇô Returns `MAXIMUM_PATHS`.

## Control Flow Notes
**Per-tick flow:**
1. `reset_paths()` ΓåÆ clears all paths
2. NPC AI ΓåÆ calls `new_path()` to request a route
3. NPC movement ΓåÆ calls `move_along_path()` per frame to traverse waypoints
4. NPC death/goal ΓåÆ calls `delete_path()` to free the slot

Flood-fill cost calculations are delegated to the caller via function pointer; default cost is polygon area.

## External Dependencies
- **map.h** ΓÇô Polygon/line/endpoint accessors; `find_shared_line()`, `get_line_data()`, `get_endpoint_data()`
- **flood_map.h** ΓÇô `flood_map()`, `reverse_flood_map()`, `flood_depth()`, `choose_random_flood_node()`; `cost_proc_ptr` typedef
- **dynamic_limits.h** ΓÇô `get_dynamic_limit()`
- **cseries.h** ΓÇô Utilities (`obj_set`, `objlist_copy`), assertions
- Standard C/C++ ΓÇô `<string.h>` (memcmp), `<limits.h>` (INT32_MAX), `<stdlib.h>`
