# Source_Files/RenderMain/RenderSortPoly.h

## File Purpose
Defines a class for sorting polygons into depth-order for rendering. Converts a visibility tree into sorted nodes, organizing polygons and their clipping windows for efficient rendering from the camera's viewpoint.

## Core Responsibilities
- Sort map polygons into depth order using visibility tree data
- Build clipping windows for viewport culling and scissoring
- Calculate vertical clip bounds for line segments
- Maintain mapping from polygon indices to sorted render nodes
- Accumulate endpoint and line clip data during sorting
- Manage resizable collections of sorted nodes and clipping data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sorted_node_data` | struct | Represents a sorted polygon node with interior/exterior render objects and clipping window |
| `RenderSortPolyClass` | class | Main container for polygon sorting state and operations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `polygon_index_to_sorted_node` | `vector<sorted_node_data *>` | member | Maps map polygon indices to sorted node pointers (only valid if polygon is visible) |
| `SortedNodes` | `vector<sorted_node_data>` | member | Growable list of sorted polygon nodes; resized in `initialize_sorted_render_tree()` and `sort_render_tree()` |
| `AccumulatedEndpointClips` | `vector<endpoint_clip_data *>` | member | Accumulated endpoint clip data pointers used during `build_clipping_windows()` |
| `AccumulatedLineClips` | `vector<line_clip_data *>` | member | Accumulated line clip data pointers used during `build_clipping_windows()` |
| `view` | `view_data *` | member | Pointer to camera/view configuration |
| `RVPtr` | `RenderVisTreeClass *` | member | Pointer to visibility tree (contains polygon visibility data) |

## Key Functions / Methods

### sort_render_tree
- **Signature:** `void sort_render_tree()`
- **Purpose:** Performs depth-order sorting of polygons for rendering
- **Inputs:** None (uses member `RVPtr` visibility tree and `view`)
- **Outputs/Return:** None (modifies `SortedNodes` and related members)
- **Side effects:** Resizes and populates `SortedNodes`, updates `polygon_index_to_sorted_node` mapping
- **Calls:** `initialize_sorted_render_tree()`, `build_clipping_windows()`
- **Notes:** Called per-frame to regenerate sorted polygon list

### Resize
- **Signature:** `void Resize(size_t NumPolygons)`
- **Purpose:** Lazy resizing of internal data structures to accommodate map size
- **Inputs:** `NumPolygons` ΓÇö max polygon count in map
- **Outputs/Return:** None
- **Side effects:** Resizes all member vectors (`SortedNodes`, `polygon_index_to_sorted_node`, clip accumulation lists)
- **Calls:** (allocation/vector operations only)
- **Notes:** Resizing is lazy; only expands, does not shrink

### RenderSortPolyClass (constructor)
- **Signature:** `RenderSortPolyClass()`
- **Purpose:** Initialize an empty sorter instance
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Constructs all member vectors in empty state
- **Calls:** None
- **Notes:** Caller must set `view` and `RVPtr` before calling `sort_render_tree()`

## Control Flow Notes
This class is invoked during the main render pipeline after `RenderVisTreeClass::build_render_tree()` produces visibility data. It converts that visibility tree into a depth-sorted list of nodes ready for rasterization. The sorting respects the camera's view cone, viewport clipping, and polygon visibility.

## External Dependencies
- `#include <vector>` ΓÇö STL vector for dynamic arrays
- `#include "world.h"` ΓÇö world coordinate types and macros
- `#include "render.h"` ΓÇö `view_data` struct and render flags
- `#include "RenderVisTree.h"` ΓÇö `node_data`, `clipping_window_data`, `endpoint_clip_data`, `line_clip_data`, `RenderVisTreeClass`
- Symbols defined elsewhere: `render_object_data` (referenced but not defined in bundled headers)
