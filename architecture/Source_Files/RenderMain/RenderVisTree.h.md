# Source_Files/RenderMain/RenderVisTree.h

## File Purpose
Defines `RenderVisTreeClass`, which constructs and manages a hierarchical visibility tree for determining which polygons and surfaces are visible from a camera viewpoint. This is a core data structure for the engine's rendering pipeline, originally derived from Marathon's render.c.

## Core Responsibilities
- Build and maintain the render visibility tree through recursive polygon traversal
- Manage viewport clipping boundaries (left, right, top, bottom edges)
- Track screen-space visibility flags for endpoints, lines, and polygons
- Calculate and store clipping information for geometry crossing screen edges
- Provide dynamic resizing of internal data structures (endpoints, lines, clipping data)
- Support polygon sorting trees for depth ordering

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `node_data` | struct | Tree node representing a polygon in the visibility hierarchy; includes clipping effects from parent, sibling/child pointers, and polygon-sort tree links |
| `line_clip_data` | struct | Clipping boundary information for a line (x bounds, top/bottom vectors and screen-space y-coordinates) |
| `endpoint_clip_data` | struct | Clipping boundary information for an endpoint (x coordinate, world-space vector, flags for left/right clipping) |
| `clipping_window_data` | struct | Describes a clipping window with left/right and top/bottom boundaries; linked-list structure |
| `RenderVisTreeClass` | class | Main visibility tree manager; aggregates node list, clipping windows, and visibility flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `POINTER_DATA` | typedef | global | Alias for `byte*` to enable correct pointer arithmetic |
| `CROSSPROD_TYPE` | typedef | global | Alias for `double` for large cross-product calculations |
| State enums (`_looking_for_first_nonzero_vertex`, etc.) | enum | file-static | Named constants for `next_polygon_along_line()` state machine |
| Clipping flag enums (`_clip_left`, `_clip_right`, etc.) | enum | file-static | Bit flags indicating active clipping sides |
| Clip index enums (`indexLEFT_SIDE_OF_SCREEN`, etc.) | enum | file-static | Named constants for screen-edge clip indices |

## Key Functions / Methods

### build_render_tree
- **Signature:** `void build_render_tree()`
- **Purpose:** Constructs the visibility tree by recursively traversing polygons visible from the camera, building a hierarchy of clipped visibility regions.
- **Inputs:** None (uses internal state: `view`, `PolygonQueue`, clipping data)
- **Outputs/Return:** None (modifies `Nodes`, clipping windows, visibility flags)
- **Side effects:** Populates `Nodes` (deque), updates `EndpointClips`, `LineClips`, `ClippingWindows`, and render flags via indices
- **Calls:** `initialize_polygon_queue()`, `initialize_render_tree()`, `initialize_clip_data()`, `cast_render_ray()`, `next_polygon_along_line()`, `calculate_line_clipping_information()`, `calculate_endpoint_clipping_information()`
- **Notes:** Entry point for tree construction; uses polygon queue to process visible geometry in breadth-first order.

### cast_render_ray
- **Signature:** `void cast_render_ray(long_vector2d *_vector, short endpoint_index, node_data* parent, short bias)`
- **Purpose:** Casts a visibility ray from a screen-edge clipping point through a polygon to find vertices and edges that cross viewport boundaries.
- **Inputs:** `_vector` (direction vector), `endpoint_index` (clipping point index), `parent` (parent tree node), `bias` (tie-breaking for ambiguous cases)
- **Outputs/Return:** None (modifies tree and clipping data structures)
- **Side effects:** Adds child nodes to parent; creates clipping data entries; modifies `Nodes` deque
- **Calls:** `next_polygon_along_line()`, `decide_where_vertex_leads()`, `calculate_endpoint_clipping_information()`, `calculate_line_clipping_information()`
- **Notes:** Uses parent node pointer (not index) to avoid stale-pointer bugs; traverses polygon edges until ray exits view cone.

### Resize
- **Signature:** `void Resize(size_t NumEndpoints, size_t NumLines)`
- **Purpose:** Lazily allocates or resizes internal dynamic containers (`EndpointClips`, `LineClips`, `ClippingWindows`, `line_clip_indexes`) to accommodate map geometry.
- **Inputs:** `NumEndpoints` (number of map endpoints), `NumLines` (number of map lines)
- **Outputs/Return:** None (adjusts vector capacities)
- **Side effects:** Resizes `line_clip_indexes` translation table and clipping data vectors
- **Calls:** Standard `std::vector` methods (`resize`)
- **Notes:** Called once per level load; allows tree to handle maps of varying sizes.

### initialize_clip_data
- **Signature:** `void initialize_clip_data()`
- **Purpose:** Clears and rebuilds clipping boundaries (left, right, top, bottom screen edges).
- **Inputs:** None (uses `view` member)
- **Outputs/Return:** None (populates `EndpointClips`, `LineClips`, `ClippingWindows`)
- **Side effects:** Wipes and reconstructs clipping data from view parameters
- **Calls:** (internal)
- **Notes:** Called at start of each visibility tree construction.

## Control Flow Notes
The visibility tree integrates into the render frame as follows:
1. **Initialization (per frame):** `build_render_tree()` is called with camera position/orientation.
2. **Visibility computation:** Builds `Nodes` tree starting from viewpoint polygon; traverses to adjacent polygons via ray-casting from screen edges.
3. **Clipping:** As polygons are added, `EndpointClips` and `LineClips` store transformed screen-edge boundaries for efficient clipping during rasterization.
4. **Rendering:** Downstream rendering code uses the tree and clipping data to draw visible surfaces with correct viewport clipping.

Polygon sort tree (PS_Greater, PS_Less, PS_Shared) is maintained in parallel for depth sorting.

## External Dependencies
- **Standard library:** `<deque>`, `<vector>` (STL containers)
- **Engine headers:**
  - `map.h` ΓÇö world geometry (polygon, line, endpoint data structures; `view_data` defined elsewhere)
  - `render.h` ΓÇö `view_data` (camera/viewport state)
- **Implicit:** Type definitions from included headers (world coordinates, vectors, fixed-point math)
