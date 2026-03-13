# Source_Files/RenderMain/RenderPlaceObjs.cpp - Enhanced Analysis

## Architectural Role

This file is the **object placement stage** of Aleph One's rendering pipeline, bridging the visibility system and the rasterizer. After `RenderVisTree` determines which polygons are visible, `RenderPlaceObjs` converts game objects into renderer-ready structures with proper screen projection, depth sorting, and clipping windows. It's uniquely responsible for a **polygon-spanning problem**: objects cross polygon boundaries, so they must be associated with multiple nodes in the depth tree and have clipping windows aggregated from all contributing polygons.

## Key Cross-References

### Incoming (who depends on this file)

- **`RenderVisTree` (RenderVisTree.cpp)**: Provides sorted polygon list (`SortedNodes`) that this file iterates
- **`RenderSortPoly` (RenderSortPoly.cpp)**: Consumes the depth-sorted render objects placed here; uses polygon clipping windows to clip rasterization
- **Main render loop** (`render.cpp`): Calls `build_render_object_list()` to populate the render tree each frame
- **Software/OpenGL rasterizers** (Rasterizer_SW.h, Rasterizer_OGL.h, Rasterizer_Shader.h): Consume `render_object_data` structures with projected rectangles, textures, and clipping info

### Outgoing (what this file depends on)

- **`GameWorld` subsystem**:
  - `map.h/cpp`: Polygon geometry, object/light/media queries
  - `player.h`: Current player index for chase-cam opacity filtering
  - `lightsource.h`: Floor/ceiling intensity lookup per polygon
  - `ephemera.h`: Temporary visual objects (particles, sprites)
  - `media.h`: Liquid height and relative positioning
  
- **`OGL_Setup.h/cpp`**: 3D model data lookup and bounding-box projection (OpenGL-conditional code)
- **`ChaseCam.h`**: Opacity override for the player object in chase-cam mode
- **`RenderSortPoly.h`**: `sorted_node_data` structure and depth-tree navigation
- **`preferences.h`**: Ephemera quality setting to conditionally include particle effects

## Design Patterns & Rationale

### 1. **Vector Reallocation Pointer Fixup**
When `RenderObjects` or `ClippingWindows` vectors reallocate, a sophisticated pointer-patching scheme maintains linked-list integrity:
```cpp
POINTER_DATA OldROPointer = POINTER_CAST(RenderObjects.data());
render_object = &RenderObjects.emplace_back();
POINTER_DATA NewROPointer = POINTER_CAST(RenderObjects.data());
if (NewROPointer != OldROPointer) {
    // Patch all next_object pointers in render objects
    // Patch interior_objects/exterior_objects in sorted nodes
}
```

**Why this pattern:**
- Avoids allocating a separate pointer-indirection layer (would hurt cache locality)
- Allows recursive parasitic object building without invalidating parent pointers
- Enables single-pass construction of object chains

**Modern contrast:** Modern engines use handle-based indirection (`ObjectID`, entity IDs) to decouple object identity from memory location, or use stable allocators that never move existing objects.

---

### 2. **Polygon-Spanning Via Ray Casting**
The `build_base_node_list()` function determines which polygon nodes an object crosses using a 2D ray-casting approach with parametric line interpolation. This is **unique to this architecture**:
- Objects are projected as 2D rectangles
- The system casts rays from the object center left and right until hitting polygon boundaries
- It computes the exact intersection point (clipping point) where the object edge crosses each polygon edge
- Stores a list of base nodes in left-to-right order

**Why this design:**
- Maintains compatibility with Marathon's 2D polygon-based world model
- Naturally handles height-based clipping (viewer above/below certain polygon boundaries)
- Produces accurate clipping windows at polygon transitions without over-clipping
- Avoids expensive 3D spatial data structures; traversal is O(polygon count)

**Modern contrast:** Modern engines typically use AABB/frustum culling with spatial acceleration (BVH, quadtree, grid). The polygon-graph traversal here is elegant but would be slow for large worlds.

---

### 3. **Clipping Windows Aggregation**
Multiple polygon nodes contribute clipping windows to a single render object. The `build_aggregate_render_object_clipping_window()` function:
- Merges overlapping windows from all base nodes
- Clips windows to the object's screen-space extent
- Computes a bounding y-clip from top and bottom of all windows

**Rationale:**
- Objects that span polygons may need to avoid rendering in regions where intermediate polygons block the view
- Avoids rendering "behind" a wall that divides two visible polygons
- Maintains depth coherence across polygon boundaries

**Architectural implication:** This tight coupling of visibility ΓåÆ clipping ΓåÆ rasterization is characteristic of the original Marathon engine. Modern engines separate these concerns: depth-of-field/z-buffer handles occlusion, frustum culling/shadow mapping handles visibility.

---

### 4. **Depth-Sorting via Intersection Testing**
Objects are inserted into a depth-sorted linked list by testing bounding-box intersection with existing objects:
- New object intersects existing objects ΓåÆ find a shallower and deeper bracket
- Adjust the desired sorted node based on these brackets
- Insert between them to maintain sorted order

**Why not a Z-buffer:**
- Marathon's rendering predates widespread Z-buffer adoption
- Software rasterizer needs back-to-front painter's algorithm for transfer modes (transparency, tints, fades)
- Allows per-polygon depth ordering, not just per-object

**Performance implication:** O(n┬▓) insertion for n objects, but with max ~72 objects (MAXIMUM_RENDER_OBJECTS), this is acceptable.

---

### 5. **3D Model Bounding-Box Projection**
When a 3D model replaces a sprite, `FindProjectedBoundingBox()` projects the model's AABB onto the sprite rectangle coordinate system:
- Rotates 8 bbox corners by the object's facing angle
- Projects via perspective division: `screen_x = center_x + (world_y * world_to_screen_x) / depth`
- Computes multiple distance references: `Farthest` (max depth), `ProjDistance` (centroid), `DistanceRef` (frame animation), `LightDepth` (miner's-light midpoint)

**Architectural insight:** 3D models are **not** truly 3D in the rendering pipelineΓÇöthey're projected to 2D and rasterized like sprites. The model's lighting is pre-baked and direction-aware (miner's light), but it's still fundamentally a 2D screen rectangle with texture and depth.

**Modern contrast:** Modern engines render 3D models directly via vertex shaders; projection happens in hardware. This approach bridges legacy 2D sprites and new 3D assets without rewriting the rasterizer.

---

## Data Flow Through This File

**Input (per frame):**
1. `SortedNodes` from visibility pass (back-to-front visible polygons)
2. Game objects and ephemera linked to each polygon
3. View parameters (camera position, FOV, projection matrix)
4. Current player object index (for chase-cam opacity)

**Transformation:**
1. Iterate polygons in reverse (back-to-front)
2. For each object in polygon:
   - `build_render_object()`: Compute screen projection, shape info, depth, transfer mode, opacity
   - `build_base_node_list()`: Determine which polygons the object spans
   - `build_aggregate_render_object_clipping_window()`: Merge clipping from all base nodes
   - `sort_render_object_into_tree()`: Insert into depth-sorted linked list

3. Recursive parasitic object handling: rebuild render objects for attached objects using relative origins

**Output:**
1. `RenderObjects` vector: All render-ready objects in a flattened linked-list structure
2. Each `render_object_data` has:
   - Screen rectangle with clamped x0/x1/y0/y1
   - Depth for painter's algorithm ordering
   - Texture, shading tables, transfer mode
   - Liquid height and media clipping
   - 3D model info if applicable (ModelPtr, ModelSequence, LightDepth, LightDirection)
   - Clipping windows from polygon boundaries
3. Sorted nodes updated with `interior_objects`/`exterior_objects` head pointers for rasterization

---

## Learning Notes

### What This Engine Does Differently
1. **2D Polygon Graph Foundation**: The entire visibility/clipping/sorting pipeline is built on 2D polygon adjacency, not 3D spatial structures. This is very fast for 2D-like levels but limits 3D scenery.

2. **Painter's Algorithm Instead of Z-Buffer**: Objects are sorted back-to-front, enabling complex transfer modes (wobble, fade, transparency) that per-pixel Z-buffering would complicate.

3. **Lighting Baked Per-Object**: Floor/ceiling intensity is computed per object, not fetched from a global light map. This maintains dynamic lighting but limits real-time light complexity.

4. **Sprite-Centric Projection**: Even 3D models are projected to sprite rectangles. This maintains compatibility with the sprite animation system and transfer modes, but limits true 3D rendering fidelity.

5. **Polygon-Relative Coordinates**: Objects store positions and dimensions relative to polygon space, with clipping windows computed dynamicallyΓÇöno pre-built light maps or shadow maps.

### Idiomatic Patterns
- **Fixed-point arithmetic for projection** to avoid floating-point rounding errors in integer screen coordinates
- **Integer-only ray casting** for polygon spanning (cross products, parametric interpolation)
- **Linked-list tree structure** for depth ordering, with pointer patching for vector reallocation
- **Recursive parasitic objects** enabling visual hierarchy (weapons on player, etc.)

---

## Potential Issues

1. **Pointer Fixup Brittle**: The vector reallocation pointer-patching is correct but fragile. A missing fixup site (e.g., if a new container of pointers is added) will silently corrupt memory. Modern approach: use handles or stable allocators.

2. **O(n┬▓) Depth Insertion**: Sorting ~72 objects by testing intersection with all prior objects is acceptable but not scalable. For large worlds with many objects, this becomes a bottleneck.

3. **No Early Culling of Off-Screen Objects**: Objects are built even if completely off-screen; clipping happens in the rasterizer. Early screen-space bounds tests could reduce work.

4. **Tight Coupling of Visibility & Clipping**: The polygon-graph traversal in `build_base_node_list()` re-traces visibility decisions already made by `RenderVisTree`. This redundancy is minor but suggests the two systems could be more unified.

5. **Light Intensity Lookup Per Object**: Calling `get_light_intensity()` for every object incurs a small per-object overhead. Caching polygon lighting in the visibility pass would be more efficient.
