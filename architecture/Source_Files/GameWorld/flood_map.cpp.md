# Source_Files/GameWorld/flood_map.cpp

## File Purpose
Implements a polygon-based graph search (flood fill) algorithm for pathfinding in the game world. Supports multiple search strategies (best-first, breadth-first) to expand polygons incrementally and retrieve paths by backtracking through parent pointers.

## Core Responsibilities
- Allocate and manage search tree memory (nodes and visited polygon tracking)
- Perform incremental graph expansion with configurable cost functions and search modes
- Track the path from source to any reached polygon via parent node pointers
- Support path retrieval via backward walk from destination to source
- Provide utility to select random expanded nodes with optional directional bias

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `node_data` | struct | 16-byte search node: flags, parent index, polygon index, cost, depth, user flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `node_count` | `short` | static | Number of nodes currently in the search tree |
| `last_node_index_expanded` | `short` | static | Index of the most recently expanded node; used for backtracking and random selection |
| `nodes` | `node_data*` | static | Array of search nodes (max 255) |
| `visited_polygons` | `short*` | static | Array indexing polygon ΓåÆ node; tracks whether a polygon has been visited |

## Key Functions / Methods

### allocate_flood_map_memory
- **Signature:** `void allocate_flood_map_memory(void)`
- **Purpose:** Allocate or reallocate search tree memory; called every time a map is loaded.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Deallocates and reallocates `nodes` (255 max) and `visited_polygons` arrays.
- **Calls:** `new`, `delete` (C++ memory management).
- **Notes:** Reentrant; safe to call multiple times. Clears prior allocations.

### flood_map
- **Signature:** `short flood_map(short first_polygon_index, int32 maximum_cost, cost_proc_ptr cost_proc, short flood_mode, void *caller_data)`
- **Purpose:** Main graph search function. Initializes search on first call (`first_polygon_index != NONE`); on subsequent calls, expands one unexpanded node with cost below `maximum_cost`.
- **Inputs:**
  - `first_polygon_index`: Starting polygon (or `NONE` to continue).
  - `maximum_cost`: Cost threshold; only nodes cheaper are expanded.
  - `cost_proc`: Function pointer to compute edge costs (NULL ΓåÆ polygon area).
  - `flood_mode`: Search strategy (_best_first, _breadth_first, _flagged_breadth_first, _depth_first).
  - `caller_data`: Mode-specific data (flags pointer for flagged breadth-first).
- **Outputs/Return:** Polygon index of next expanded node; `NONE` if search exhausted.
- **Side effects:** Modifies `node_count`, `last_node_index_expanded`, `nodes`, `visited_polygons`; calls cost function.
- **Calls:** `add_node()`, `get_polygon_data()`, `objlist_set()`.
- **Notes:** 
  - Best-first guarantees optimal paths but slower on large domains.
  - Breadth-first faster but explores more nodes.
  - Depth-first unsupported (asserts).
  - Zero/negative-cost edges not added.
  - Adjacent detached or already-expanded polygons skipped.

### reverse_flood_map
- **Signature:** `short reverse_flood_map(void)`
- **Purpose:** Walk backward through the path from last expanded node to source via parent pointers.
- **Inputs:** None (uses `last_node_index_expanded`).
- **Outputs/Return:** Polygon index at prior step; `NONE` when path exhausted.
- **Side effects:** Updates `last_node_index_expanded`.
- **Calls:** None.
- **Notes:** Used after `flood_map()` returns a destination to retrieve the full path.

### flood_depth
- **Signature:** `short flood_depth(void)`
- **Purpose:** Return the search depth (polygon count) of the last expanded node.
- **Inputs:** None.
- **Outputs/Return:** Depth (0 if `last_node_index_expanded == NONE`).
- **Side effects:** None.
- **Calls:** None.

### choose_random_flood_node
- **Signature:** `void choose_random_flood_node(world_vector2d *bias)`
- **Purpose:** Select a random expanded node; if `bias` provided, prefer nodes in that direction.
- **Inputs:** `bias` ΓÇö optional direction vector (pointer to 2D vector).
- **Outputs/Return:** None (sets `last_node_index_expanded`).
- **Side effects:** Updates `last_node_index_expanded`.
- **Calls:** `global_random()`, `find_center_of_polygon()`.
- **Notes:** Requires at least 1 node in tree. May loop infinitely on very small maps. Uses dot product to check alignment with bias.

### add_node (private)
- **Signature:** `static void add_node(short parent_node_index, short polygon_index, short depth, int32 cost, int32 user_flags)`
- **Purpose:** Add a new node or replace an existing higher-cost node in the search tree.
- **Inputs:** Parent node index, polygon index, depth, cost, user flags.
- **Outputs/Return:** None.
- **Side effects:** Updates `nodes`, `visited_polygons[polygon_index]`, increments `node_count`.
- **Calls:** None.
- **Notes:** 
  - Silently fails if `node_count >= MAXIMUM_FLOOD_NODES`.
  - Replaces existing node only if it's unexpanded and has higher cost.
  - Sets visited mark so duplicate checks work.

## Control Flow Notes
Typical sequence:
1. **Initialization:** `allocate_flood_map_memory()` when map loads.
2. **Search init:** `flood_map(start_polygon, max_cost, cost_fn, mode, data)` to seed the queue.
3. **Incremental expansion:** Call `flood_map(NONE, ...)` repeatedly until it returns `NONE`.
4. **Path retrieval:** `reverse_flood_map()` called repeatedly to walk backward from destination.
5. **Query:** `flood_depth()` to check expansion distance.

The file does not directly participate in frame/render loops; it is utility code called by pathfinding and AI systems.

## External Dependencies
- **Includes:** `cseries.h`, `map.h`, `flood_map.h`, standard C++ (`<string.h>`, `<stdlib.h>`, `<limits.h>`).
- **Defined elsewhere:**
  - `polygon_data`, `node_data` (map.h)
  - `get_polygon_data()` ΓÇö accessor for polygon structures.
  - `find_center_of_polygon()` ΓÇö computes polygon center.
  - `global_random()` ΓÇö RNG.
  - `objlist_set()` ΓÇö macro to fill array.
  - `MAXIMUM_POLYGONS_PER_MAP`, `MAXIMUM_FLOOD_NODES` ΓÇö limits.
  - `dynamic_world` ΓÇö global world state.
- **Namespace:** `alephone::flood_map` (internal struct defined here; functions are module-scoped).
