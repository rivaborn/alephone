# Source_Files/RenderMain/RenderVisTree.cpp

## File Purpose

Implements the rendering visibility tree class for determining which polygons are visible from the player's viewpoint. Core functionality includes ray casting through the 2D map to build a hierarchical visibility tree, managing polygon visibility, and calculating screen-space clipping information for the renderer.

## Core Responsibilities

- **Build visibility tree**: Construct a tree of visible polygons starting from the viewpoint polygon
- **Ray casting**: Cast rays from view edges and polygon endpoints to find visible adjacent polygons
- **Polygon traversal**: Walk along rays crossing polygon boundaries to determine visibility
- **Clipping calculation**: Compute line and endpoint clipping data for elevation and transparency
- **Endpoint transformation**: Transform world coordinates to screen space for visibility tests
- **Queue management**: Maintain and process a queue of polygons needing visibility checks
- **Automap tracking**: Record visible polygons and lines for automap display

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `node_data` | struct | Tree node representing a visible polygon with clipping info and parent/sibling/child links |
| `endpoint_clip_data` | struct | Screen-space clipping vector for an endpoint |
| `line_clip_data` | struct | Screen-space clipping bounds and vectors for a line |
| `clipping_window_data` | struct | 2D clipping window for rendering |

## Global / File-Static State

None (all state encapsulated in `RenderVisTreeClass`).

## Key Functions / Methods

### build_render_tree()
- **Signature**: `void RenderVisTreeClass::build_render_tree()`
- **Purpose**: Main entry point; constructs the entire visibility tree from the player's viewpoint
- **Inputs**: None (uses `view` member pointer)
- **Outputs/Return**: None (modifies internal `Nodes` and `PolygonQueue`)
- **Side effects**: Populates visibility tree, sets render flags on polygons/endpoints/lines, updates polygon queue
- **Calls**: `initialize_polygon_queue()`, `initialize_render_tree()`, `initialize_clip_data()`, `cast_render_ray()`, `PUSH_POLYGON_INDEX()`
- **Notes**: Entry polygon index comes from `view->origin_polygon_index`; processes left/right view edges first, then queue-driven BFS of adjacent polygons

### cast_render_ray()
- **Signature**: `void RenderVisTreeClass::cast_render_ray(long_vector2d *_vector, short endpoint_index, node_data* parent, short bias)`
- **Purpose**: Recursively trace a ray through polygons, building the visibility tree as it crosses boundaries
- **Inputs**: Ray vector in world space, optional endpoint being targeted, parent node, bias for traversal direction
- **Outputs/Return**: None (builds tree via `parent->children`/`siblings`)
- **Side effects**: Allocates new `node_data` entries, sets clipping info on nodes, updates polygon sort tree
- **Calls**: `next_polygon_along_line()`, `calculate_line_clipping_information()`, `calculate_endpoint_clipping_information()` (recursively calls itself for split rays)
- **Notes**: Uses parent pointer (not index after fix) to avoid stale-pointer bugs; splits rays when `_split_render_ray` flag is set; maintains polygon sort BST (`PS_Greater/Less/Shared`)

### next_polygon_along_line()
- **Signature**: `uint16 RenderVisTreeClass::next_polygon_along_line(short *polygon_index, world_point2d *origin, long_vector2d *_vector, short *clipping_endpoint_index, short *clipping_line_index, short bias)`
- **Purpose**: Trace a ray through a single polygon to find the next polygon it exits into
- **Inputs**: Current polygon index, ray origin, ray direction vector, optional target endpoint, bias direction
- **Outputs/Return**: Clip flags (`_clip_up/_clip_down/_clip_left/_clip_right`) via output pointers; updates `polygon_index`/`clipping_line_index`
- **Side effects**: Sets render flags on visited endpoints/sides; adds polygon to automap; marks exploration polygons
- **Calls**: `PUSH_POLYGON_INDEX()`, `decide_where_vertex_leads()`, `get_line_data()`, `get_endpoint_data()`
- **Notes**: Uses finite state machine (looking for nonzero vertices) and cross-product tests to determine ray-polygon intersection; handles vertex-on-ray cases

### decide_where_vertex_leads()
- **Signature**: `uint16 RenderVisTreeClass::decide_where_vertex_leads(short *polygon_index, short *line_index, short *side_index, short endpoint_index_in_polygon_list, world_point2d *origin, long_vector2d *_vector, uint16 clip_flags, short bias)`
- **Purpose**: When a ray passes through a vertex, determine which adjacent polygon to traverse next based on bias direction
- **Inputs**: Current polygon, vertex position within polygon, ray origin/vector, bias (_clockwise, _counterclockwise, or _no_bias)
- **Outputs/Return**: Updated polygon/line/side indices; clip flags
- **Side effects**: Possibly sets `_split_render_ray` flag if bias is `_no_bias`
- **Calls**: `get_line_data()`, `get_endpoint_data()`
- **Notes**: For solid endpoints, sets `_clip_left` or `_clip_right` depending on traversal direction

### calculate_line_clipping_information()
- **Signature**: `void RenderVisTreeClass::calculate_line_clipping_information(short line_index, uint16 clip_flags)`
- **Purpose**: Compute screen-space clipping bounds and vectors for a line that clips the view (elevation change)
- **Inputs**: Line index, clip flags (`_clip_up/_clip_down`)
- **Outputs/Return**: None (appends to `LineClips` vector)
- **Side effects**: Allocates new `line_clip_data`, sets `_line_has_clip_data` flag on line
- **Calls**: `get_line_data()`, `get_endpoint_data()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()`
- **Notes**: Handles long-distance views by converting to 64-bit coordinates; calculates top/bottom clip vectors and screen y-coordinates

### calculate_endpoint_clipping_information()
- **Signature**: `short RenderVisTreeClass::calculate_endpoint_clipping_information(short endpoint_index, uint16 clip_flags)`
- **Purpose**: Compute screen-space clipping vector for an endpoint that clips the view
- **Inputs**: Endpoint index, clip flags (`_clip_left/_clip_right`)
- **Outputs/Return**: Index into `EndpointClips` vector (or `NONE` if not transformed)
- **Side effects**: Appends to `EndpointClips` vector, sets `_endpoint_has_clip_data` flag
- **Calls**: None (inline calculations)
- **Notes**: Only processes endpoints that were previously transformed; validates clip flags

### PUSH_POLYGON_INDEX()
- **Signature**: `void RenderVisTreeClass::PUSH_POLYGON_INDEX(short polygon_index)`
- **Purpose**: Add a polygon to the processing queue if not already marked visible
- **Inputs**: Polygon index
- **Outputs/Return**: None (appends to `PolygonQueue`)
- **Side effects**: Sets `_polygon_is_visible` flag; grows `PolygonQueue` if needed
- **Calls**: `TEST_RENDER_FLAG()`, `SET_RENDER_FLAG()`
- **Notes**: Deduplicates via visibility flag

## Control Flow Notes

**Initialization phase** (in `build_render_tree()`):
1. Clear polygon queue
2. Create root node for origin polygon
3. Initialize clipping data (default screen edge clips)

**Ray casting phase**:
1. Cast rays from left and right view edges toward the first adjacent polygons
2. Recursively cast rays at endpoints of each processed polygon

**Queue-driven phase**:
1. While polygon queue is not empty:
   - Pop polygon
   - For each endpoint not yet visited:
     - Transform to screen space
     - Test if in view cone (cross products)
     - Cast ray at the endpoint

**Clipping phase** (on-demand):
- Line clipping info is calculated when crossing an elevation line
- Endpoint clipping info is calculated when targeting an opaque endpoint

## External Dependencies

- **cseries.h**: Utility types and macros
- **map.h**: Polygon/line/endpoint/side/view data structures; macros for flag testing
- **render.h**: `view_data` structure definition
- **STL**: `<vector>`, `<deque>` for growable containers
- **Defined elsewhere**: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()` (all map accessors)
