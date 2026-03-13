# Source_Files/RenderMain/RenderPlaceObjs.h

## File Purpose
Defines a class that organizes game objects (inhabitants) into a visibility-sorted tree for correct rendering order. Acts as a bridge between game objects and the rendering subsystem, managing object placement, clipping, and depth sorting. Part of the Aleph One game engine rendering pipeline.

## Core Responsibilities
- Build render objects from game objects with lighting and opacity parameters
- Sort render objects into a visibility tree (polygon-based spatial structure)
- Calculate and manage per-object clipping windows for screen rendering
- Maintain linked lists of objects per sorted polygon node (interior/exterior)
- Integrate with visibility tree and polygon sorting systems
- Rescale shape information as needed for rendering

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `render_object_data` | struct | Wraps a game object with rendering metadata: sorted node reference, clipping window, linked-list pointer, screen rectangle, and media depth coordinate. |
| `RenderPlaceObjsClass` | class | Container and coordinator for object placement; holds lists of render objects and pointers to visibility/sorting structures. |

## Global / File-Static State
None.

## Key Functions / Methods

### build_render_object_list
- Signature: `void build_render_object_list()`
- Purpose: Public entry point; orchestrates building and sorting all render objects for the current frame.
- Inputs: (none; uses member pointers to view, visibility tree, sorted polygons)
- Outputs/Return: Updates `RenderObjects` vector and internal tree structures.
- Side effects: Modifies visibility tree and sorted polygon nodes; allocates/modifies clipping windows.
- Calls: Likely calls `initialize_render_object_list()`, `build_render_object()`, `sort_render_object_into_tree()` for each object.
- Notes: Main rendering loop entry point for object placement.

### build_render_object
- Signature: `render_object_data* build_render_object(object_data* object, _fixed floor_intensity, _fixed ceiling_intensity, float Opacity, long_point3d* origin, long_point3d* rel_origin)`
- Purpose: Construct a render object from a game object with lighting and position.
- Inputs: Game object pointer, floor/ceiling light intensities, opacity, absolute and relative origins.
- Outputs/Return: Pointer to newly created `render_object_data`.
- Side effects: May allocate clipping windows; modifies `RenderObjects` vector.
- Calls: Calls `build_aggregate_render_object_clipping_window()`, likely shape queries.
- Notes: Lighting parameters control object visibility through attenuation.

### sort_render_object_into_tree
- Signature: `void sort_render_object_into_tree(render_object_data* new_render_object, const span_data& span)`
- Purpose: Insert a render object into the visibility/polygon sorting tree.
- Inputs: Render object pointer, span metadata from visibility tree.
- Outputs/Return: (none; in-place tree modification)
- Side effects: Updates sorted polygon nodes and clipping structures.
- Calls: Calls `build_base_node_list()`, `build_aggregate_render_object_clipping_window()`.
- Notes: Determines which polygon node(s) contain the object and inserts into linked lists.

## Control Flow Notes
This class is invoked once per frame during the render phase, after visibility tree computation (`RenderVisTreeClass`) and polygon sorting (`RenderSortPolyClass`). It places objects into the depth-sorted tree, enabling correct rendering order: back-to-front with proper clipping per polygon region.

## External Dependencies
- Includes: `<vector>`, `world.h`, `interface.h`, `render.h`, `RenderSortPoly.h`
- Uses: `view_data` (view parameters), `sorted_node_data` (polygon nodes), `clipping_window_data` (screen clipping), `object_data` (game objects), `shape_information_data` (sprite metrics), `RenderVisTreeClass`, `RenderSortPolyClass` (visibility/sorting results)
