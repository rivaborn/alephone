# Source_Files/RenderOther/OverheadMap_OGL.h

## File Purpose
OpenGL-specific implementation of the overhead map (mini-map) renderer for the Aleph One game engine. Subclass of `OverheadMapClass` that provides GPU-accelerated rendering of game map elements, entities, and annotations.

## Core Responsibilities
- Implement OpenGL rendering for map polygons, lines, entities, and text
- Cache geometry data (polygons, lines, paths) for efficient batch rendering to GPU
- Render game entities (monsters, items, projectiles, checkpoints) on the overhead map
- Draw the player character with facing direction
- Support text annotation rendering with font specifications
- Implement begin/end batching lifecycle for each geometry type
- Manage path visualization for debugging/development

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `PolygonCache` | vector | Cached polygon vertex indices for batch rendering |
| `LineCache` | vector | Cached line endpoints for batch rendering |
| `PathPoints` | vector | Cached waypoints for path visualization |
| `SavedColor` | struct | Color state for current polygon batch |
| `SavedPenSize` | scalar | Line width state for current line batch |

## Global / File-Static State
None.

## Key Functions / Methods

### begin_overall / end_overall
- **Purpose:** Lifecycle hooks for overall map rendering setup and teardown
- **Calls:** None visible

### begin_polygons / draw_polygon / end_polygons / DrawCachedPolygons
- **Purpose:** Batch polygon rendering; accumulates polygons in cache, submits when `end_polygons()` or `DrawCachedPolygons()` called
- **Signature:** `draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- **Inputs:** Vertex count, vertex index array, fill color
- **Side effects:** Modifies `PolygonCache`, `SavedColor`

### begin_lines / draw_line / end_lines / DrawCachedLines
- **Purpose:** Batch line rendering; caches line segments with pen width, submits when flushed
- **Signature:** `draw_line(short *vertices, rgb_color& color, short pen_size)`
- **Inputs:** Vertex pair, color, pen width
- **Side effects:** Modifies `LineCache`, `SavedPenSize`

### draw_thing
- **Purpose:** Render single map entity (monster, item, projectile, checkpoint)
- **Signature:** `draw_thing(world_point2d& center, rgb_color& color, short shape, short radius)`
- **Inputs:** Position, color, shape type, screen radius

### draw_player
- **Purpose:** Render player icon with facing direction indicator
- **Signature:** `draw_player(world_point2d& center, angle facing, rgb_color& color, short shrink, short front, short rear, short rear_theta)`
- **Inputs:** Center position, facing angle, color, shrink factor, shape geometry parameters

### draw_text
- **Purpose:** Render text annotation with font styling and justification
- **Signature:** `draw_text(world_point2d& location, rgb_color& color, char *text, FontSpecifier& FontData, short justify)`
- **Inputs:** Location, color, text string, font spec, justification (0=left, 1=center)

### set_path_drawing / draw_path / finish_path
- **Purpose:** Path visualization API for debugging; accumulates waypoints, flushes on finish
- **Signature:** `draw_path(short step, world_point2d &location)` where step=0 for first point
- **Side effects:** Modifies `PathPoints`

## Control Flow Notes
Standard graphics pipeline batching pattern: `begin_*()` ΓåÆ multiple `draw_*()` calls ΓåÆ `end_*()` or `DrawCached*()` to flush to GPU. This allows accumulating geometry before expensive GPU submission.

## External Dependencies
- `<vector>` (STL containers)
- `OverheadMapRenderer.h` (base class `OverheadMapClass`, config data structures, type definitions)
- Engine types: `world_point2d`, `angle`, `rgb_color`, `FontSpecifier` (defined elsewhere)
