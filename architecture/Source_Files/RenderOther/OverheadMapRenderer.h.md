# Source_Files/RenderOther/OverheadMapRenderer.h

## File Purpose
Base class and configuration structures for overhead map (minimap/automap) rendering. Provides virtual methods for graphics-API-agnostic rendering and defines styling data (colors, fonts, shapes) for all automap elements including polygons, lines, entities, and annotations.

## Core Responsibilities
- Define configuration structures (`OvhdMap_CfgDataStruct`) for automap appearance (polygon/line colors, entity shapes, fonts, paths)
- Provide virtual base class (`OverheadMapClass`) for graphics-API-specific renderer implementations
- Define rendering pipeline with virtual hooks (begin/end polygons, lines, things, text, paths)
- Handle coordinate transformation for map viewport rendering
- Support false automap generation (checkpoint/saved-game preview modes)
- Expose static accessor methods for vertex data to subclasses (esp. OpenGL)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `line_definition` | struct | Color and pen sizes (per scale) for overhead map lines |
| `thing_definition` | struct | Color, shape type, and radii (per scale) for entities/items/projectiles on map |
| `entity_definition` | struct | Geometry offsets (front, rear, rear_theta) for player entity display |
| `annotation_definition` | struct | Font and color set for map text annotations at different scales |
| `map_name_definition` | struct | Font, color, and vertical offset for level name display |
| `OvhdMap_CfgDataStruct` | struct | Master configuration: all colors, shapes, fonts, visibility flags for map rendering |
| `OverheadMapClass` | class | Virtual base class for rendering implementations; holds config pointer and pure virtual methods |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `saved_automap_lines` | `byte*` | private static | Saved automap line visibility for false automap (checkpoint/preview) generation |
| `saved_automap_polygons` | `byte*` | private static | Saved automap polygon visibility for false automap generation |

## Key Functions / Methods

### Render
- Signature: `void Render(overhead_map_data& Control)`
- Purpose: Main entry point; initiates rendering of the overhead map with specified configuration
- Inputs: `overhead_map_data` struct containing mode, scale, viewport origin, dimensions
- Outputs/Return: None (void); side effects drive rendering
- Side effects: Calls virtual methods to render polygons, lines, things, text, and paths
- Calls: `begin_overall()`, `begin_polygons()`, `end_polygons()`, `begin_lines()`, `end_lines()`, `begin_things()`, `end_things()`, `draw_polygon()` (overloads), `draw_line()` (overloads), `draw_thing()` (overloads), `draw_player()`, `draw_text()`, path drawing methods, map name drawing
- Notes: Driven by configuration in `ConfigPtr`; virtual methods allow subclass-specific graphics APIs

### Virtual rendering stage methods
- `begin_overall()`, `end_overall()` ΓÇô Wrap entire rendering session
- `begin_polygons()`, `end_polygons()` ΓÇô Delimit polygon batch
- `begin_lines()`, `end_lines()` ΓÇô Delimit line batch
- `begin_things()`, `end_things()` ΓÇô Delimit entity/item/projectile batch
- All have empty default implementations; subclasses override as needed for state management

### draw_polygon (virtual)
- Signature: `virtual void draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- Purpose: Render a single filled polygon on the map
- Inputs: vertex count, vertex indices, RGB color
- Outputs/Return: None
- Notes: Pure virtual; must be overridden by subclass with graphics API calls

### draw_line (virtual)
- Signature: `virtual void draw_line(short *vertices, rgb_color& color, short pen_size)`
- Purpose: Render a single line segment on the map
- Inputs: endpoint indices (2-element array), RGB color, pen width
- Outputs/Return: None
- Notes: Pure virtual; overridden by subclass

### draw_thing (virtual)
- Signature: `virtual void draw_thing(world_point2d& center, rgb_color& color, short shape, short radius)`
- Purpose: Render a single entity/item/projectile marker on the map
- Inputs: center position, color, shape constant (rectangle/circle), radius
- Outputs/Return: None
- Notes: Pure virtual; shape differentiates between `_rectangle_thing` and `_circle_thing`

### draw_player (virtual)
- Signature: `virtual void draw_player(world_point2d& center, angle facing, rgb_color& color, short shrink, short front, short rear, short rear_theta)`
- Purpose: Render the player entity with direction/orientation indicator
- Inputs: center, facing angle, color, shrink factor, entity geometry offsets
- Outputs/Return: None
- Notes: Pure virtual

### draw_text (virtual)
- Signature: `virtual void draw_text(world_point2d& location, rgb_color& color, char *text, FontSpecifier& FontData, short justify)`
- Purpose: Render text (annotations, map name) at specified location
- Inputs: location, color, text string, font specifier, justification mode (`_justify_left` or `_justify_center`)
- Outputs/Return: None
- Notes: Pure virtual

### set_path_drawing (virtual)
- Signature: `virtual void set_path_drawing(rgb_color& color)` and overload `void set_path_drawing()` (private inline)
- Purpose: Set color for subsequent path drawing
- Inputs: RGB color
- Outputs/Return: None
- Notes: Overload uses `ConfigPtr->path_color`

### draw_path (virtual)
- Signature: `virtual void draw_path(short step, world_point2d& location)`
- Purpose: Render one waypoint/segment of a path (called iteratively)
- Inputs: step index (0 = first point), waypoint location
- Outputs/Return: None
- Notes: Pure virtual; used for both initial and subsequent path vertices

### finish_path (virtual)
- Signature: `virtual void finish_path()`
- Purpose: Signal end of path drawing; clean up if needed
- Outputs/Return: None
- Notes: Pure virtual

### GetVertex (static inline)
- Signature: `static world_point2d& GetVertex(short index)`
- Purpose: Accessor for transformed endpoint data
- Inputs: endpoint index
- Outputs/Return: Reference to transformed vertex (`endpoint_data::transformed`)
- Notes: For subclass convenience (especially OpenGL); defined elsewhere via `get_endpoint_data()`

### GetFirstVertex, GetVertexStride (static inline)
- Purpose: Convenience accessors for batch vertex data retrieval
- Outputs/Return: Pointer to first vertex; stride (size of `endpoint_data`)
- Notes: For OpenGL array rendering

## Control Flow Notes
- **Init**: `OverheadMapClass` constructor initializes `ConfigPtr` to NULL; subclass must set before calling `Render()`
- **Main path**: `Render()` receives `overhead_map_data` specifying viewport (scale, origin, dimensions, mode) and calls virtual rendering methods to draw all map elements
- **False automap**: Auxiliary methods (`generate_false_automap()`, `false_automap_cost_proc()`, `replace_real_automap()`) support special modes like checkpoint/saved-game previews where the automap is rebuilt using a cost function
- **Coordinate transformation**: `transform_endpoints_for_overhead_map()` pre-processes all map geometry into viewport-relative coordinates
- **Rendering order**: Polygons ΓåÆ lines ΓåÆ things ΓåÆ text/annotations ΓåÆ path (top-to-bottom in draw calls)

## External Dependencies
- **Notable includes**: 
  - `cseries.h` ΓÇô Core types (rgb_color, RGBColor, angle)
  - `world.h` ΓÇô World geometry (world_point2d, world_distance, angle)
  - `map.h` ΓÇô Map structures (endpoint_data, line_data via `get_line_data()`, etc.)
  - `monsters.h` ΓÇô Monster types
  - `shape_descriptors.h` ΓÇô Shape descriptor macros
  - `shell.h` ΓÇô Platform I/O (_get_player_color)
  - `FontHandler.h` ΓÇô FontSpecifier for text rendering
- **Defined elsewhere**:
  - `get_endpoint_data()`, `get_line_data()` ΓÇô Accessors to map geometry
  - `overhead_map_data` ΓÇô Viewport specification (defined in `overhead_map.h`)
  - `endpoint_data.transformed` ΓÇô Cached viewport-space vertex coordinates
  - `_render_overhead_map()` ΓÇô Likely calls `Render()` on a concrete subclass instance
