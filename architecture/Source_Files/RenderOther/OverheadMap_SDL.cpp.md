# Source_Files/RenderOther/OverheadMap_SDL.cpp

## File Purpose
Implements SDL-based rendering for the overhead (minimap) display in Aleph One. Provides concrete drawing methods for map elements: geometry, objects, players, text labels, and path trails, inheriting from `OverheadMapClass`.

## Core Responsibilities
- Convert map vertex indices to world coordinates and rasterize polygons
- Render lines, game objects (rectangles/circles), and player indicators
- Draw text annotations with font and justification control
- Manage incremental path drawing for player trails
- Map RGB color values to SDL pixel format

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | struct (external) | Color channels (red, green, blue shifted by 8 bits) |
| `world_point2d` | struct (external) | 2D coordinate in world space |
| `angle` | typedef (external) | Player facing direction |
| `FontSpecifier` | struct (external) | Font with style flags and font info pointer |
| `SDL_Surface` | struct (SDL) | Drawing target surface |
| `SDL_Rect` | struct (SDL) | Axis-aligned rectangle for rasterization |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `draw_surface` | `SDL_Surface*` | extern (screen_sdl.cpp) | Active rendering target |
| `vertex_array` | `world_point2d*` | static (draw_polygon) | Reusable vertex buffer for polygon conversion |
| `max_vertices` | `int` | static (draw_polygon) | Current capacity of vertex_array |
| `path_pixel` | `uint32` | member (private) | Cached SDL pixel color for path segments |
| `path_point` | `world_point2d` | member (private) | Last endpoint of path trail |

## Key Functions / Methods

### draw_polygon
- Signature: `void draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- Purpose: Rasterize a filled polygon onto the overhead map
- Inputs: vertex count, array of vertex indices (into GetVertex lookup), RGB color
- Outputs/Return: None (side effect: pixels drawn)
- Side effects: Allocates/reallocates `vertex_array` if capacity exceeded; calls `::draw_polygon` (external)
- Calls: `GetVertex()` (base class), `SDL_MapRGB()`, `::draw_polygon()`
- Notes: Dynamically grows vertex buffer; no bounds checking on input count

### draw_line
- Signature: `void draw_line(short *vertices, rgb_color &color, short pen_size)`
- Purpose: Render a line segment between two vertices
- Inputs: vertices[0] and vertices[1] indices, RGB color, pen width
- Outputs/Return: None
- Side effects: Draws to `draw_surface`
- Calls: `GetVertex()`, `SDL_MapRGB()`, `::draw_line()`
- Notes: Expects exactly 2 vertices; no validation

### draw_thing
- Signature: `void draw_thing(world_point2d &center, rgb_color &color, short shape, short radius)`
- Purpose: Draw a game object as a rectangle or 8-sided circle approximation
- Inputs: center position, color, shape type (`_rectangle_thing` or `_circle_thing`), radius
- Outputs/Return: None
- Side effects: Rasterizes filled rect or circle outline to `draw_surface`
- Calls: `SDL_MapRGB()`, `SDL_FillRect()`, `::draw_line()`
- Notes: Circle uses 8 line segments; radius scaled by 0.75/1.5; no other shape types handled

### draw_player
- Signature: `void draw_player(world_point2d &center, angle facing, rgb_color &color, short shrink, short front, short rear, short rear_theta)`
- Purpose: Render player as directional triangle (isosceles: one front point, two rear)
- Inputs: center position, facing angle, RGB color, shrink divisor (for rear points), front/rear distances, rear angle spread
- Outputs/Return: None
- Side effects: Computes triangle vertices, draws to `draw_surface`
- Calls: `translate_point2d()` (twice), `normalize_angle()`, `SDL_MapRGB()`, `::draw_polygon()`
- Notes: Front point is at facing angle; two rear points are placed symmetrically rear_theta apart

### draw_text
- Signature: `void draw_text(world_point2d &location, rgb_color &color, char *text, FontSpecifier& FontData, short justify)`
- Purpose: Render text label on the overhead map
- Inputs: location, RGB color, text string, font spec (Info + Style), justify flag
- Outputs/Return: None
- Side effects: Adjusts x position for centering, draws text via `::draw_text()`
- Calls: `text_width()`, `SDL_MapRGB()`, `::draw_text()`
- Notes: Only `_justify_center` centering is implemented; `_justify_center` false draws at location.x directly

### set_path_drawing
- Signature: `void set_path_drawing(rgb_color &color)`
- Purpose: Cache the color for subsequent path segments
- Inputs: RGB color
- Outputs/Return: None
- Side effects: Sets `path_pixel` member
- Calls: `SDL_MapRGB()`
- Notes: Must be called before first `draw_path()` call

### draw_path
- Signature: `void draw_path(short step, world_point2d &location)`
- Purpose: Incrementally draw path segments; maintains state across calls
- Inputs: step flag (0 = first point / reset; non-zero = draw segment from last point), current location
- Outputs/Return: None
- Side effects: Draws line from `path_point` to `location` if step != 0; updates `path_point`
- Calls: `::draw_line()` (if step != 0)
- Notes: `step` acts as a reset flag; subsequent calls draw segments between consecutive locations

## Control Flow Notes
These methods are called by the parent `OverheadMapClass` during map rendering. Typical sequences:
- **Geometry**: `draw_polygon()` for floor/sector outlines
- **Objects**: `draw_thing()` for scenery, items, monsters
- **Player**: `draw_player()` for each player
- **Paths**: `set_path_drawing()` once per player, then `draw_path(0, ...)` then `draw_path(1, ...)` for each segment
- **Text**: `draw_text()` for map annotations

All drawing is immediate to `draw_surface`; no buffering within this class.

## External Dependencies
- **SDL2**: `SDL_MapRGB()`, `SDL_FillRect()`, `SDL_Surface`, `SDL_Rect`
- **Aleph One core**: `world_point2d`, `rgb_color`, `angle`, `FontSpecifier`
- **External functions**: `::draw_polygon()`, `::draw_line()`, `::draw_text()` (from screen_drawing.h; rasterization backends)
- **Base class**: `GetVertex()` (OverheadMapClass), `text_width()` (font utility)
- **Global**: `draw_surface` (extern SDL_Surface* from screen_sdl.cpp)
