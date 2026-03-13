# Source_Files/RenderMain/RenderSortPoly.h - Enhanced Analysis

## Architectural Role

This class bridges the **visibility determination phase** (RenderVisTree) and **rasterization phase** by converting an unordered visibility tree into a depth-sorted linear rendering list. It's a critical adapter in the 3D pipeline that computes all scissor rectangles and object-ordering information *once per frame* before rasterization begins, enabling the downstream software and GPU rasterizers to work with pre-sorted, pre-clipped geometry. The interior/exterior object separation implements painter's algorithm supportΓÇöobjects are renderable in two passes relative to each polygon's primary surface.

## Key Cross-References

### Incoming (who depends on this file)
- **render.cpp** ΓÇö Orchestration layer calls `RenderSortPolyClass::sort_render_tree()` and `Resize()` per frame after visibility is determined
- **RenderRasterize.h/cpp** ΓÇö Consumes `SortedNodes` vector during polygon rasterization; reads clipping windows for scissor setup
- **Rasterizer_*.h** backends ΓÇö Iterate over sorted nodes in order, respecting pre-computed clipping regions

### Outgoing (what this file depends on)
- **RenderVisTree.h/cpp** ΓÇö `RVPtr` member holds the visibility tree output; `build_render_tree()` must complete before `sort_render_tree()` runs
- **render.h** ΓÇö Provides `view_data` struct (camera/viewport state); `render_object_data` struct referenced (defined in render subsystem)
- **world.h** ΓÇö Provides map geometry types used indirectly through `node_data` structures from visibility tree

## Design Patterns & Rationale

### Accumulator Pattern
`AccumulatedEndpointClips` and `AccumulatedLineClips` are populated during `build_clipping_windows()` ΓÇö this lazy accumulation defers clip computation to window-building time rather than computing per-polygon, reducing redundant calculations.

### Lazy Resizing
The `Resize(size_t NumPolygons)` design only allocates *if* the map is larger than previously seen. This was critical in 2000 (when this was written) to avoid worst-case allocations for small maps.

### STL Modernization (2000)
Header comments explicitly note migration from `GrowableList`/`ResizableList` to `std::vector`ΓÇöa sign of C++ idiom adoption during the Aleph One fork era. The commented-out `NodeAliases` static reflects lessons learned: the BSP-like visibility tree structure already provides polygon ordering, making explicit alias tracking redundant.

### Painter's Algorithm Support
The dual `interior_objects` / `exterior_objects` lists in `sorted_node_data` enable rendering in two passes:
1. Objects behind the polygon (interior)
2. The polygon surface
3. Objects in front of the polygon (exterior)

This avoids depth-sort failures for complex object/polygon relationships in a 2.5D-like environment.

## Data Flow Through This File

```
RenderVisTree outputs (node_data tree)
    Γåô
sort_render_tree() processes visibility tree recursively
    Γö£ΓåÆ initialize_sorted_render_tree() (setup)
    Γö£ΓåÆ build_clipping_windows(node_data*) for each node
    Γöé  ΓööΓåÆ accumulates endpoint/line clips
    Γöé  ΓööΓåÆ calculate_vertical_clip_data() computes scissor bounds
    ΓööΓåÆ populates SortedNodes[] & polygon_index_to_sorted_node[] mapping
    Γåô
RenderRasterize/Rasterizer backends
    Γö£ΓåÆ iterate SortedNodes in order
    Γö£ΓåÆ read clipping_windows for scissor rectangles
    ΓööΓåÆ dispatch interior_objects, polygon surface, then exterior_objects
```

The `polygon_index_to_sorted_node` mapping is marked "only valid if `_polygon_is_visible`"ΓÇörasterizers must check visibility before dereferencing.

## Learning Notes

- **Visibility ΓåÆ Sorting Separation**: Unlike naive renderers that sort-while-testing-visibility, Aleph One decouples these: RenderVisTree determines *what* is visible; RenderSortPoly determines *order* and *scissor clips*. This enables swap-in rasterizer backends without visibility logic duplication.
- **Era Idiom**: The 2000-era hand-rolled memory management (growable lists, lazy allocation) contrasts sharply with modern engines using generational GC or frame arenas. This codebase predates widespread STL adoption in game engines.
- **Pre-computed Clipping**: Rather than clipping during rasterization, clipping windows are pre-computed here. This is a **memory/bandwidth tradeoff**ΓÇöstoring clip rectangles costs memory but eliminates per-pixel clip tests during rasterization.
- **View Member Not Parameter**: The comment notes `view_data *view` was promoted from function parameter to member. This eliminated redundant parameter passing through recursive visibility tree traversalΓÇöa common optimization in deep call stacks.

## Potential Issues

- **No validation** that `view` and `RVPtr` are set before `sort_render_tree()` is called. If either is null, the sort will crash. Modern engines would assert or use RAII initialization.
- **No per-frame cleanup**: Accumulated clips (`AccumulatedEndpointClips`, `AccumulatedLineClips`) are presumably cleared/reused across frames, but the public interface doesn't expose that responsibilityΓÇöcoupling with `sort_render_tree()` internals.
- **Commented-out NodeAliases note** suggests a prior bug or complexity that was resolved. No commit context available, but the comment indicates depth-ordering was previously a source of edge cases.
