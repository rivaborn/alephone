# Source_Files/RenderMain/RenderPlaceObjs.cpp

## File Purpose
Implements object placement and depth-sorting for rendering. Converts game objects into renderer-ready data structures with proper projection, visibility culling, clipping windows, and depth ordering within a polygon tree.

## Core Responsibilities
- Build sorted render-object list from world objects per polygon
- Create render objects with projected screen coordinates and shape/transfer-mode data
- Determine which polygons (nodes) an object spans via line-crossing analysis
- Sort objects into depth tree based on intersection testing
- Aggregate clipping windows from multiple nodes
- Handle 3D model bounding-box projection and scaling
- Support parasitic objects (objects attached to host objects)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `RenderPlaceObjsClass::span_data` | struct | Tracks base nodes and screen-space span of a render object |
| `span_data::base_node_data` | struct | One polygon node + right edge point of span within that node |
| `render_object_data` | struct | (from RenderPlaceObjs.h) Linked-list node for a renderable object with clipping and depth info |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MAXIMUM_RENDER_OBJECTS` | #define (72) | static | Reserved size hint for render-object vector |
| `NUMBER_OF_SCALED_VALUES` | #define (6) | static | Count of scaled shape dimensions |
| `FindProjectedBoundingBox` | static function | static | Helper to project 3D model bounding boxes |

## Key Functions / Methods

### RenderPlaceObjsClass::RenderPlaceObjsClass (constructor)
- **Signature:** `RenderPlaceObjsClass()`
- **Purpose:** Initialize member pointers and pre-allocate render-object vector.
- **Inputs:** None.
- **Outputs/Return:** Object constructed.
- **Side effects:** Sets `view`, `RVPtr`, `RSPtr` to NULL; reserves 72 slots in `RenderObjects`.
- **Calls:** `vector::reserve`

### RenderPlaceObjsClass::build_render_object_list
- **Signature:** `void build_render_object_list()`
- **Purpose:** Traverse sorted polygons in reverse order and accumulate all objects into the render list.
- **Inputs:** `view`, `RVPtr`, `RSPtr` must be set; current map polygons and objects.
- **Outputs/Return:** Populates `RenderObjects`.
- **Side effects:** Clears render list, iterates world and calls `add_object_to_sorted_nodes` per object; applies chase-cam opacity.
- **Calls:** `initialize_render_object_list`, `add_object_to_sorted_nodes`, `get_polygon_data`, `get_object_data`, `get_light_intensity`, `GetChaseCamData`, `get_polygon_ephemera`, `get_ephemera_data`.

### RenderPlaceObjsClass::build_render_object
- **Signature:** `render_object_data* build_render_object(object_data*, _fixed floor_intensity, _fixed ceiling_intensity, float Opacity, long_point3d* origin, long_point3d* rel_origin)`
- **Purpose:** Create a single render object from game object; handle projection, clipping, transfer modes, 3D models, and parasites.
- **Inputs:** Object data, lighting, optional transformed position, relative origin for parasites.
- **Outputs/Return:** Pointer to new render object (or NULL if invisible/out-of-range/no shape).
- **Side effects:** 
  - Appends to `RenderObjects`; may reallocate and invalidate pointers in render-object and sorted-node lists.
  - Computes screen-space rectangle, depth, opacity, model info.
  - Recursively calls self for parasitic objects; updates pointer after reallocation.
- **Calls:** `transform_overflow_point2d`, `extended_get_shape_information`, `OGL_GetModelData`, `FindProjectedBoundingBox`, `get_object_shape_and_transfer_mode`, `rescale_shape_information`, `extended_get_shape_bitmap_and_shading_table`, `instantiate_rectangle_transfer_mode`, `get_media_data`, `get_polygon_data`, `get_dynamic_limit`.
- **Notes:** 
  - Handles both sprites and 3D models; bounding-box projection used for models.
  - Pointer arithmetic used to fix up linked-list pointers after vector reallocation.
  - Chase-cam opacity applied per object.
  - Clamps projected values to short range.

### RenderPlaceObjsClass::sort_render_object_into_tree
- **Signature:** `void sort_render_object_into_tree(render_object_data* new_render_object, const span_data& span)`
- **Purpose:** Insert new render object into depth-sorted linked list, maintaining depth order within desired node.
- **Inputs:** Newly created render object(s), span info (which nodes it crosses).
- **Outputs/Return:** Object linked into sorted tree.
- **Side effects:** Modifies `next_object` pointers in render-object list; sets `node` field on new object(s).
- **Calls:** Cross-product comparisons on rectangle bounds and depth.
- **Notes:** 
  - Finds two intersecting objects (shallower and deeper) to bracket new object.
  - Adjusts desired node based on intersecting objects' nodes.
  - Sorts within node or at node boundaries to preserve depth order.

### RenderPlaceObjsClass::build_base_node_list
- **Signature:** `span_data build_base_node_list(const render_object_data* render_object, short origin_polygon_index)`
- **Purpose:** Determine which polygons (sorted nodes) the object spans by cross-product ray casting.
- **Inputs:** Render object (for bounds), origin polygon index.
- **Outputs/Return:** `span_data` with base-node list and left/right extent points.
- **Side effects:** Traverses polygon adjacency graph via `scan_toward` lambda.
- **Calls:** `get_polygon_data`, `get_endpoint_data`, `cross_product_k`, `normalize_angle`, `cosine_table`, `sine_table`.
- **Notes:** 
  - Uses scan-toward lambda to walk polygon boundaries left and right from origin.
  - Calculates clipping points along polygon edges via parametric interpolation.
  - Handles visibility checks and height-based clipping (viewer above/below).

### RenderPlaceObjsClass::build_aggregate_render_object_clipping_window
- **Signature:** `void build_aggregate_render_object_clipping_window(render_object_data* render_object, const span_data& span)`
- **Purpose:** Aggregate clipping windows from all base nodes into a cohesive set for the object.
- **Inputs:** Render object(s), span data.
- **Outputs/Return:** Assigns clipping windows to all objects in chain.
- **Side effects:** May append to `ClippingWindows` vector; reallocates and fixes pointers if needed.
- **Calls:** `eye_vec_toward` lambda (2D cross-product in view space), pointer arithmetic for reallocation fixup.
- **Notes:** 
  - Merges overlapping node windows and clips to object extent.
  - Computes bounding y-clip from top and bottom of all contributing windows.
  - Similar pointer-fixup logic as `build_render_object`.

### RenderPlaceObjsClass::add_object_to_sorted_nodes
- **Signature:** `bool add_object_to_sorted_nodes(object_data*, _fixed floor_intensity, _fixed ceiling_intensity, float Opacity)`
- **Purpose:** High-level entry point: build render object, find its span, clip windows, and sort into tree.
- **Inputs:** Game object, lighting, opacity.
- **Outputs/Return:** `true` if object added; `false` if render object could not be created.
- **Side effects:** Calls sequence of `build_*` methods in order.
- **Calls:** `build_render_object`, `build_base_node_list`, `build_aggregate_render_object_clipping_window`, `sort_render_object_into_tree`.

### RenderPlaceObjsClass::rescale_shape_information
- **Signature:** `shape_information_data* rescale_shape_information(shape_information_data* unscaled, shape_information_data* scaled, uint16 flags)`
- **Purpose:** Scale shape dimensions (world_left, world_right, etc.) by 1.25├ù or 0.5├ù based on object flags.
- **Inputs:** Unscaled shape info, flags (`_object_is_enlarged`, `_object_is_tiny`).
- **Outputs/Return:** Pointer to scaled (or unscaled if no flags) shape info.
- **Side effects:** Modifies `scaled` struct in-place if flags set; otherwise returns `unscaled`.
- **Notes:** Trivial helper; no complex logic.

### FindProjectedBoundingBox (static function)
- **Signature:** `void FindProjectedBoundingBox(GLfloat BoundingBox[2][3], long_point3d& TransformedPosition, GLfloat Scale, short RelativeAngle, shape_information_data& ShapeInfo, short DepthType, int& Farthest, int& ProjDistance, int& DistanceRef, int& LightDepth, GLfloat *Direction)`
- **Purpose:** Project a 3D model's bounding box onto screen space; compute depth references and miner's-light direction.
- **Inputs:** 3D bbox (2 corners in 3D), object position/scale/rotation, shape info struct, depth-type flag.
- **Outputs/Return:** Updates `ShapeInfo` with projected world bounds; sets `Farthest`, `ProjDistance`, `DistanceRef`, `LightDepth`, `Direction`.
- **Side effects:** Computes 8-vertex expanded bbox, projects via angle rotation and perspective division.
- **Calls:** `normalize_angle`, `cosine_table`, `sine_table`, `isqrt`, `std::clamp`.
- **Notes:** 
  - Expands 2-corner bbox into 8 vertices via rotation scaling.
  - Depth-type selects reference distance (max, min, or center).
  - LightDepth (midpoint between closest and bbox center) used for miner's-light effect.
  - Direction normalized via fast reciprocal square root.

## Control Flow Notes
**InitΓåÆBuildΓåÆSort pipeline:**
1. `build_render_object_list()` iterates sorted polygons (back-to-front in world).
2. For each object, calls `add_object_to_sorted_nodes()`.
3. That sequence calls `build_render_object()` ΓåÆ `build_base_node_list()` ΓåÆ `build_aggregate_render_object_clipping_window()` ΓåÆ `sort_render_object_into_tree()`.
4. Result: render objects are in depth order within their nodes, ready for rendering.

**Reallocation safety:** When `RenderObjects` or `ClippingWindows` vectors reallocate, all affected pointers in both lists and sorted nodes are patched via pointer arithmetic.

## External Dependencies
- `map.h`, `world.h` ΓÇö polygon/object/light structs, accessor functions
- `lightsource.h` ΓÇö light intensity lookup
- `media.h` ΓÇö media height queries
- `render.h`, `RenderSortPoly.h` ΓÇö render tree and sorted-node structures
- `OGL_Setup.h` ΓÇö 3D model data and queries
- `ChaseCam.h` ΓÇö chase-cam opacity
- `player.h` ΓÇö current player index
- `ephemera.h` ΓÇö ephemera object queries
- `preferences.h` ΓÇö graphics preferences (ephemera quality)
- STL: `<vector>`, `<algorithm>`, `<boost/container/small_vector.hpp>`
