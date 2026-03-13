# Source_Files/RenderOther/OverheadMap_SDL.h

## File Purpose
SDL-specific subclass of OverheadMapClass that implements the abstract rendering interface using Simple DirectMedia Layer. Provides concrete implementations for drawing polygons, lines, entities, player indicators, text, and path visualization on the overhead (mini) map.

## Core Responsibilities
- Implement SDL rendering pipeline for overhead map polygons
- Render line segments (walls, elevation changes, control panels)
- Draw game entities (monsters, items, projectiles, checkpoints)
- Render player representation with directional facing indicators
- Render text annotations at map locations with font specification
- Manage path drawing state and incremental path rendering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OverheadMap_SDL_Class | class | SDL-based renderer inheriting from OverheadMapClass |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| path_pixel | uint32 | private member | Cached pixel color value for path rendering |
| path_point | world_point2d | private member | Previous path point location for incremental drawing |

## Key Functions / Methods

### draw_polygon
- **Signature:** `void draw_polygon(short vertex_count, short *vertices, rgb_color &color)`
- **Purpose:** Render a filled or outlined polygon on the overhead map
- **Inputs:** vertex count, array of vertex indices, color
- **Outputs/Return:** None (void)
- **Side effects:** Writes to SDL rendering surface
- **Calls:** Not visible in this file (implementation in .cpp)
- **Notes:** Virtual override; vertices are indices into the transformed vertex list

### draw_line
- **Signature:** `void draw_line(short *vertices, rgb_color &color, short pen_size)`
- **Purpose:** Render a line segment (wall edge, elevation change, etc.)
- **Inputs:** Two vertex indices, color, pen width
- **Outputs/Return:** None
- **Side effects:** Writes to SDL rendering surface
- **Calls:** Not visible in this file

### draw_thing
- **Signature:** `void draw_thing(world_point2d &center, rgb_color &color, short shape, short radius)`
- **Purpose:** Render a game entity (monster, item, projectile, checkpoint)
- **Inputs:** World position, color, shape code, radius in map units
- **Outputs/Return:** None
- **Side effects:** Writes to SDL rendering surface
- **Calls:** Not visible in this file

### draw_player
- **Signature:** `void draw_player(world_point2d &center, angle facing, rgb_color &color, short shrink, short front, short rear, short rear_theta)`
- **Purpose:** Render player representation with directional facing indicator
- **Inputs:** Player position, facing angle, color, shrink scale, front/rear shape params
- **Outputs/Return:** None
- **Side effects:** Writes to SDL rendering surface
- **Calls:** Not visible in this file
- **Notes:** The front/rear/rear_theta parameters define the player indicator shape

### draw_text
- **Signature:** `void draw_text(world_point2d &location, rgb_color &color, char *text, FontSpecifier& FontData, short justify)`
- **Purpose:** Render text annotation at a location on the map
- **Inputs:** Text position, color, text string, font specification, justification (left/center)
- **Outputs/Return:** None
- **Side effects:** Writes to SDL rendering surface
- **Calls:** Not visible in this file

### set_path_drawing
- **Signature:** `void set_path_drawing(rgb_color &color)`
- **Purpose:** Initialize path drawing state with color
- **Inputs:** Path color (from configuration)
- **Outputs/Return:** None
- **Side effects:** Caches `path_pixel` color value
- **Calls:** Not visible in this file

### draw_path
- **Signature:** `void draw_path(short step, world_point2d &location)`
- **Purpose:** Draw incremental path segments (called multiple times per path)
- **Inputs:** Step index (0 for first point, increments thereafter), current location
- **Outputs/Return:** None
- **Side effects:** Writes to SDL rendering surface; updates `path_point`
- **Calls:** Not visible in this file
- **Notes:** Draws line from previous point; step=0 initializes without drawing

## Control Flow Notes
This class is instantiated as the active renderer for overhead map display. The base class `OverheadMapClass::Render()` drives the pipeline by calling `begin_*()`, drawing methods, then `end_*()` hooks in sequence: polygons ΓåÆ lines ΓåÆ things ΓåÆ player ΓåÆ annotations/paths. The SDL implementation translates these abstract calls into SDL drawing commands.

## External Dependencies
- **Inherits from:** `OverheadMapClass` (OverheadMapRenderer.h)
- **Type dependencies:** `rgb_color`, `world_point2d`, `angle` (from world.h), `FontSpecifier` (from FontHandler.h)
- **Not included in this header:** SDL itself (assumed in .cpp implementation)
