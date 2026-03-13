# Source_Files/RenderMain/RenderSortPoly.cpp

## File Purpose

Implements depth-based polygon sorting for the rendering pipeline. Transforms an unordered visibility tree into an ordered list of polygons with computed clipping windows. Part of the Aleph One game engine's 3D rendering system.

## Core Responsibilities

- Sort polygons into depth order by recursively traversing the visibility tree
- Build clipping windows (left/right/top/bottom screen bounds) for each sorted polygon
- Calculate vertical clipping data (top and bottom edge constraints) for rendering
- Manage dynamic reallocation of sorted nodes and clipping windows with pointer fixup
- Maintain mapping between polygon indices and sorted node pointers

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `RenderSortPolyClass` | class | Manages polygon sorting and clipping window generation |
| `sorted_node_data` | struct (from header) | A sorted polygon with its render objects and clipping windows |
| `node_data` | struct (external) | Node in visibility tree; has polygon index, parent, siblings, clipping info |
| `clipping_window_data` | struct (external) | Screen-space clipping bounds (x0, x1, y0, y1 with vectors) |

## Global / File-Static State

None.

## Key Functions / Methods

### RenderSortPolyClass::RenderSortPolyClass()
- **Signature:** `RenderSortPolyClass()`
- **Purpose:** Initialize class state; pre-allocate vector capacity
- **Inputs:** None
- **Outputs/Return:** Initializes member vectors and pointers
- **Side effects:** Sets `view=NULL`, `RVPtr=NULL`, reserves capacity for `SortedNodes` (128) and clip lists (64 each)
- **Calls:** `std::vector::reserve()`
- **Notes:** Idiot-proofing via null initialization

### RenderSortPolyClass::Resize()
- **Signature:** `void Resize(size_t NumPolygons)`
- **Purpose:** Pre-size the polygon-to-sorted-node index mapping
- **Inputs:** `NumPolygons` ΓÇô number of polygons in the map
- **Outputs/Return:** Resizes `polygon_index_to_sorted_node` vector
- **Side effects:** Reallocates vector; clears old entries
- **Calls:** `std::vector::resize()`
- **Notes:** Called once per level initialization

### RenderSortPolyClass::initialize_sorted_render_tree()
- **Signature:** `void initialize_sorted_render_tree()`
- **Purpose:** Clear the sorted node list before sorting
- **Inputs:** None (reads `SortedNodes`)
- **Outputs/Return:** None
- **Side effects:** Clears `SortedNodes` vector
- **Calls:** `std::vector::clear()`
- **Notes:** Called at start of `sort_render_tree()`

### RenderSortPolyClass::sort_render_tree()
- **Signature:** `void sort_render_tree()`
- **Purpose:** Main depth-sorting algorithm; walks visibility tree leaf-to-root, building sorted node list
- **Inputs:** `RVPtr->Nodes` (visibility tree), `view` (view context)
- **Outputs/Return:** Populates `SortedNodes` in depth order; updates `polygon_index_to_sorted_node` mapping
- **Side effects:** Modifies `SortedNodes`, `polygon_index_to_sorted_node`, `RVPtr->ClippingWindows`; calls `build_clipping_windows()` for each polygon
- **Calls:** `initialize_sorted_render_tree()`, `build_clipping_windows()`, `SortedNodes.emplace_back()`, `memmove()` (for vector shifts)
- **Notes:** Uses a binary search tree (PS_Greater/PS_Less) to locate all node instances with the same polygon index; processes the PS_Shared chain of aliases. Handles vector reallocation and pointer fixup when `SortedNodes` grows.

### RenderSortPolyClass::build_clipping_windows()
- **Signature:** `clipping_window_data *build_clipping_windows(node_data *ChainBegin)`
- **Purpose:** Compute and build clipping windows (left/right/top/bottom bounds) for a given polygon and its parents' clipping info
- **Inputs:** `ChainBegin` ΓÇô start of PS_Shared chain for one polygon's nodes
- **Outputs/Return:** Pointer to first clipping window added (or `nullptr` if none)
- **Side effects:** Accumulates endpoint and line clips from the node and all parents; inserts clipping windows into `RVPtr->ClippingWindows`; updates `polygon_index_to_sorted_node` if vector reallocates; calls `calculate_vertical_clip_data()`
- **Calls:** `get_polygon_data()`, `build_clipping_windows()` (recursive via chain), `calculate_vertical_clip_data()`, `ClippingWindows.emplace_back()`, `memmove()`
- **Notes:** Walks the node hierarchy (parents) to gather all clipping constraints. Handles vector reallocation with pointer fixup. Maintains a linked list of clipping windows.

### RenderSortPolyClass::calculate_vertical_clip_data()
- **Signature:** `void calculate_vertical_clip_data(line_clip_data **accumulated_line_clips, size_t accumulated_line_clip_count, clipping_window_data *window, short x0, short x1)`
- **Purpose:** Determine top and bottom screen clipping lines for a horizontal range [x0, x1]
- **Inputs:** Array of accumulated line clips, count, window bounds, x0/x1 horizontal range
- **Outputs/Return:** Fills `window->top`, `window->y0`, `window->bottom`, `window->y1`
- **Side effects:** None (read-only on line clips, write to window)
- **Calls:** None (pure computation)
- **Notes:** Finds the highest top clip and lowest bottom clip covering the range; iterates by segment. Does not require line clips to be sorted.

## Control Flow Notes

This class is part of the rendering pipeline's visibility-to-sorted-geometry stage. `sort_render_tree()` is the main entry point, called during scene preparation. It consumes the visibility tree (from `RVPtr`) and produces a sorted node list in depth order, along with pre-computed clipping windows for screen-space scissoring during rasterization.

The algorithm decomposes the tree destructively: it pulls leaf polygons off one at a time and adds them to `SortedNodes`. For each polygon, it computes which screen regions are valid for rendering by examining all parent-node clipping constraints.

## External Dependencies

- **Includes:** `cseries.h` (base utilities, macros), `map.h` (map/polygon data), `RenderSortPoly.h` (header)
- **Standard library:** `<string.h>` (memmove), `<limits.h>` (SHRT_MAX, SHRT_MIN)
- **External symbols:** 
  - `get_polygon_data(index)` ΓÇô from map subsystem
  - `RenderVisTreeClass::NodeList`, `node_data` ΓÇô visibility tree structure
  - `clipping_window_data`, `endpoint_clip_data`, `line_clip_data` ΓÇô from render.h
  - `view_data` ΓÇô screen/camera state (from render.h)
  - Macros: `TEST_RENDER_FLAG`, `POINTER_CAST`, `POINTER_DATA`, `vassert`, `csprintf`, `NONE`, `MAX`, `MIN`
