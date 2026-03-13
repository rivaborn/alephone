# Source_Files/RenderMain/RenderVisTree.h - Enhanced Analysis

## Architectural Role
This class is the **visibility determination engine** for Aleph One's rendering pipeline. It bridges the gap between raw world geometry (`map.h`) and rasterization by computing which polygons are visible from the camera and calculating screen-space clipping regions. Its output feeds directly into `RenderSortPoly.h/cpp` for depth ordering and `RenderRasterize.h` for final polygon rasterization. The tree is fundamentally a **ray-casting data structure**: it shoots rays from screen edges through map geometry to determine visibility boundaries, then stores the resulting hierarchy in `Nodes` with parallel clipping data (`EndpointClips`, `LineClips`) that downstream code uses to clip geometry to the viewport.

## Key Cross-References

### Incoming (who depends on this file)
- **`RenderMain/render.cpp`** ΓÇö Calls `build_render_tree()` once per frame after camera setup; consumes populated `Nodes`, clipping windows, and visibility flags
- **`RenderMain/RenderSortPoly.h/cpp`** ΓÇö Reads the `Nodes` hierarchy and `ClippingWindows` to perform depth-based polygon sorting and clipping window creation (parallels the tree structure with `clipping_window_data` per node)
- **`RenderMain/RenderPlaceObjs.h/cpp`** ΓÇö Uses visibility flags from `Nodes` to cull objects before projection
- **`RenderMain/RenderRasterize.h/cpp`** ΓÇö Consumes `EndpointClips` and `LineClips` translation tables for polygon clipping during rasterization
- **Software rasterizer (`scottish_textures.cpp`)** ΓÇö Uses `line_clip_indexes` translation table to map world-space lines to clip data for efficient lookup during line rasterization

### Outgoing (what this file depends on)
- **`GameWorld/map.h`** ΓÇö World geometry: `polygon_data`, `line_data`, `endpoint_data`, `world_point2d` types; spatial queries for adjacent polygons
- **`render.h`** ΓÇö `view_data` struct (camera position, orientation, viewport dimensions, FOV); used to initialize clipping planes
- **STL containers** ΓÇö `std::vector` and `std::deque` for dynamic allocation of endpoints, lines, windows, and tree nodes
- **File-local state machine enums** ΓÇö `next_polygon_along_line()` state constants; clipping flag bit masks

## Design Patterns & Rationale

### Ray-Casting Visibility
The algorithm casts rays **outward from the camera** through screen edges into the world to discover which polygons bound the visible region. Rather than frustum culling (modern approach), this edge-based ray casting is more accurate for orthographic/isometric-style projection and handles portal visibility naturallyΓÇöa polygon is visible if any ray starting from a screen edge reaches it without crossing a nearer blocking polygon.

### Dual Tree Structures
Two parallel tree relationships are maintained:
1. **Parent/sibling/children** ΓÇö Hierarchical visibility tree reflecting ray traversal order (parent = polygon that contained the ray origin)
2. **Polygon-sort tree** (PS_Greater/PS_Less/PS_Shared) ΓÇö Binary tree for depth ordering by polygon index; allows efficient sorted traversal during rendering

This dual structure decouples **visibility hierarchy** from **depth ordering**, a key optimization to avoid rebuilding the sort tree separately.

### Lazy Resizing & Translation Tables
- `Resize(NumEndpoints, NumLines)` is called once per level load, pre-allocating vectors to map world geometry counts
- `line_clip_indexes[world_line_index] ΓåÆ clip_index_in_LineClips[]` avoids linear searches during rasterization
- This **translation table pattern** trades memory for O(1) clip data lookup

### Fixed-Size Arrays Per Node
Each `node_data` has `MAXIMUM_CLIPPING_ENDPOINTS_PER_NODE` (4) and `MAXIMUM_CLIPPING_LINES_PER_NODE` (polygon vertices - 2) arrays, not vectors. This reflects the algorithm's assumption that clipping complexity is bounded by polygon geometryΓÇöa sound design for Marathon-era performance constraints but potentially fragile for highly concave polygons.

## Data Flow Through This File

1. **Input Phase** (`build_render_tree()` entry):
   - Camera state from `view` member (`view_data`: position, orientation, viewport bounds)
   - World geometry: polygon adjacency, line/endpoint coordinates from `map.h`

2. **Initialization** (`initialize_*()` calls):
   - `initialize_polygon_queue()` ΓÇö Enqueue the viewpoint polygon
   - `initialize_render_tree()` ΓÇö Clear `Nodes`, set up root node
   - `initialize_clip_data()` ΓÇö Build initial clipping planes (screen edges) stored in `EndpointClips`/`LineClips`/`ClippingWindows`

3. **Core Traversal** (breadth-first via `PolygonQueue`):
   - Dequeue polygon ΓåÆ `cast_render_ray()` for each screen edge clipping point
   - `cast_render_ray()` ΓåÆ `next_polygon_along_line()` ΓÇö Walk polygon edges until ray exits current polygon, detecting which adjacent polygon to enter
   - `next_polygon_along_line()` ΓåÆ state machine (`_looking_clockwise_for_right_vertex`, etc.) traces polygon boundary
   - Create child `node_data`, enqueue newly discovered polygon
   - For each new screen-edge crossing, call `calculate_endpoint_clipping_information()` and `calculate_line_clipping_information()` to update clipping data

4. **Output**:
   - `Nodes` ΓÇö Deque of tree nodes forming hierarchy
   - `ClippingWindows` ΓÇö Linked-list of clipping region definitions (left/right/top/bottom bounds)
   - `EndpointClips` / `LineClips` ΓÇö Screen-space clipping boundary vectors and coordinates
   - Render flags set on polygons (via indices) for downstream visibility tests

## Learning Notes

### Era-Specific Design Choices
- **Pointer-based tree** instead of modern intrusive trees or child vectors ΓÇö typical 1990s C code porting to C++; performance was paramount on fixed hardware
- **Double-precision vectors for long views** ΓÇö Comments note long-distance friendly fixes; Marathon maps could have large expanses requiring precise intersection calculation
- **M1-style exploration goal** (`mark_as_explored` flag) ΓÇö Aleph One added support for Marathon 1's older game rules alongside Marathon 2/Infinity rules; render tree customization for that mode

### Architectural Evolution Within File
- Comments track migration from **GrowableLists/ResizableLists ΓåÆ STL vectors** (Oct 2000), improving code maintainability
- **Stale-pointer bug fix** (Sep 2000): switched from storing parent pointer to storing parent index, then accessing via `Nodes[index]`; illustrates hidden dangers of pointer-based trees during container resizing
- Macros like `PUSH_POLYGON_INDEX` and `WRAP_LOW/WRAP_HIGH` suggest earlier procedural code; partial C++ification

### Idiomatic Differences
Modern engines typically use:
- **GPU frustum culling** over CPU ray casting
- **PVS (Potentially Visible Sets)** precomputed offline vs. per-frame traversal
- **Spatial hashing/BVH** over raw polygon adjacency
- **Compute shaders** for visibility on GPU

Aleph One's approach is **CPU-bound, frame-by-frame, geometrically precise** ΓÇö reflecting the 2000s software-rasterization era and Marathon's architectural legacy.

## Potential Issues

### Clipping Array Overflow Risk
`MAXIMUM_CLIPPING_ENDPOINTS_PER_NODE` and `MAXIMUM_CLIPPING_LINES_PER_NODE` are **fixed-size arrays** with no bounds checking in the class definition. If `calculate_endpoint_clipping_information()` or `calculate_line_clipping_information()` exceed these limits (possible in degenerate polygon geometry), stack corruption or silent data loss could occur. Modern code would use vectors with bounds assertions.

### Raw Pointer Tree Fragility
The `node_data` struct uses raw parent/sibling/children pointers into the `Nodes` deque. Deque reallocation (rare but possible if `Nodes` exceeds capacity and grows) could invalidate these pointers. The `Resize()` method only pre-allocates but doesn't guarantee safety if the deque grows unexpectedly. A safer design would store indices or use intrusive pointers.

### Bidirectional Lookup Gap
The `line_clip_indexes` translation table maps world line indices ΓåÆ clip indices, but there's no reverse lookup. If a clip line is modified, updating the corresponding world-space geometry requires iterating `LineClips` to find the source polygonΓÇöinefficient for dynamic geometry updates (not currently done, but a design smell for future extensibility).
