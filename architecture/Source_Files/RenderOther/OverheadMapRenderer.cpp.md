# Source_Files/RenderOther/OverheadMapRenderer.cpp

## File Purpose
Implements the base overhead-map (automap) rendering system for Marathon. Transforms world coordinates to screen space and renders game-world features (polygons, lines, entities, annotations) from a top-down view, with special support for limited-visibility checkpoint maps via flood-fill algorithms.

## Core Responsibilities
- Main render loop: coordinates overall automap drawing via `begin_*()` / `end_*()` virtual hooks
- Viewport transforms: converts world coordinates to screen space using fixed-point bit-shift scaling
- Polygon coloring: determines colors based on type (platform, water, lava, etc.), media presence, and secret status
- Line visibility: draws elevation/solid edges between polygons with proper ownership checks
- Entity rendering: draws players (with team colors), monsters, items, projectiles with visibility filtering
- Annotation/text: positions and renders map labels at appropriate scales
- Path visualization: renders AI pathfinding waypoints when enabled
- Checkpoint automap: generates/restores limited visibility maps via flood-fill cost function and state save/restore

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OverheadMapClass` | class | Base renderer (defined in header); subclassed for platform-specific rendering |
| `overhead_map_data` | struct | Control parameters: viewport origin, position, scale, rendering mode |
| `world_point2d` | struct | 2D world coordinates; transformed to screen coords and stored in endpoints |
| `polygon_data` | struct | World geometry; provides vertex indices, type, media index, transfer modes |
| `line_data` | struct | Line segment; tracks owning polygon indices and solid/variable elevation flags |
| `endpoint_data` | struct | Vertex; holds both world (`vertex`) and screen-space (`transformed`) positions |
| `platform_data` | struct | Platform entity; checked for secret/door/flooded status for color selection |
| `media_data` | struct | Liquid/hazard; provides type and height for color override logic |
| `monster_data` / `player_data` | struct | Entity data; used for team, visibility, and display type lookup |
| `map_annotation` | struct | Text label; position and type (color) for annotation rendering |
| `map_object` | struct | Saved object from level; used in checkpoint mode to mark goal location |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `saved_automap_lines` | `byte*` | private member | Backup of automap line visibility bitfield during checkpoint rendering |
| `saved_automap_polygons` | `byte*` | private member | Backup of automap polygon visibility bitfield during checkpoint rendering |
| `dynamic_world` | struct* | extern | Current game-world state; accessed for polygon/line/endpoint/object counts and data |
| `static_world` | struct* | extern | Level data; accessed for level name |
| `objects` | object_data[] | extern | Game entities array; iterated to render players, monsters, items, projectiles |
| `saved_objects` | map_object[] | extern | Initial object placements; used in checkpoint mode to find goal marker |
| `automap_lines` / `automap_polygons` | byte[] | extern | Bitfields tracking which lines/polygons are explored; temporarily swapped in checkpoint mode |
| `ConfigPtr` | OvhdMap_CfgDataStruct* | member | Configuration pointer; holds color/pen/font definitions; must be set before `Render()` |

## Key Functions / Methods

### Render
- **Signature:** `void OverheadMapClass::Render(overhead_map_data& Control)`
- **Purpose:** Main rendering entry point; orchestrates drawing of all map elements from world coordinates transformed to a screen viewport.
- **Inputs:** `Control` ΓÇö viewport parameters (origin, position, scale), rendering mode (game map vs. checkpoint map)
- **Outputs/Return:** None (renders via virtual methods `draw_polygon()`, `draw_line()`, `draw_thing()`, `draw_player()`, etc.)
- **Side effects:** 
  - Calls `begin_overall()` / `end_overall()` for platform-specific setup/teardown
  - Modifies game options flags (`GET_GAME_OPTIONS()`) based on config
  - In checkpoint mode: calls `generate_false_automap()` and `replace_real_automap()` to temporarily limit visibility
  - May call game-world accessors (`get_polygon_data()`, `get_line_data()`, etc.) which are expected not to fail
- **Calls:**
  - `begin_overall()`, `end_overall()` ΓÇö virtual hooks for platform-specific rendering setup
  - `generate_false_automap()` ΓÇö create limited automap for checkpoint mode
  - `transform_endpoints_for_overhead_map()` ΓÇö convert all vertices to screen space
  - `begin_polygons()`, `end_polygons()`, `draw_polygon()` ΓÇö polygon rendering sub-loop
  - `begin_lines()`, `end_lines()`, `draw_line()` ΓÇö line rendering sub-loop
  - `draw_annotation()` ΓÇö text label rendering
  - `set_path_drawing()`, `draw_path()`, `finish_path()` ΓÇö path visualization
  - `draw_player()`, `draw_thing()` ΓÇö entity rendering
  - `draw_map_name()` ΓÇö level name display
  - `replace_real_automap()` ΓÇö restore automap after checkpoint mode
  - Game accessors: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `get_media_data()`, `get_platform_data()`, `get_monster_data()`, `get_player_data()`, `get_next_map_annotation()`, `get_number_of_paths()`, `path_peek()`
- **Notes:**
  - Coordinate transformation: `WORLD_TO_SCREEN(world_coord, origin, scale)` computes `(world_coord - origin) >> (8 - scale)` using a fixed-point shift
  - Screen offsets: `xoff = Control.left + Control.half_width`, `yoff = Control.top + Control.half_height` center the viewport
  - Polygon color selection is complex: base color depends on type, overridden by media type if media height ΓëÑ floor; secret platforms get different color
  - Flooded platforms (via `PLATFORM_IS_FLOODED()`) use the flooded-into polygon's type color (ouch zone)
  - Line color depends on both ownership and topology: solid/variable elevation vs. height discontinuity; landscaped lines filtered out if both owners are landscape transfer mode
  - Entity visibility: monsters/items filtered by config; players only shown if omniscient or same team; blinking effect for non-projectile things using `(tick_count + i) & 8`
  - Checkpoint mode: only shows goal marker and limited visibility; no live entities rendered

### transform_endpoints_for_overhead_map
- **Signature:** `void OverheadMapClass::transform_endpoints_for_overhead_map(overhead_map_data& Control)`
- **Purpose:** Transform all world-space vertices to screen-space coordinates and mark which are on-screen; pre-process visibility based on endpoint positions.
- **Inputs:** `Control` ΓÇö viewport parameters (origin, position, dimensions, scale)
- **Outputs/Return:** None (modifies endpoint data in place; sets `_endpoint_on_automap` and `_polygon_on_automap` flags)
- **Side effects:**
  - Modifies `endpoint->transformed` for all endpoints (accessed later in `draw_polygon()`)
  - Sets state flags on endpoints and polygons via `SET_STATE_FLAG()`
- **Calls:**
  - `get_endpoint_data(i)` ΓÇö retrieve endpoint for transformation
  - `get_polygon_data(i)` ΓÇö retrieve polygon and its vertex indices
  - `TEST_STATE_FLAG()`, `SET_STATE_FLAG()` ΓÇö flag manipulation macros
- **Notes:**
  - First loop: transforms all endpoints and marks those within the clip rectangle
  - Second loop: marks polygons that have at least one visible endpoint; early-exits via `break` when first visible endpoint found
  - Clip rectangle: `left Γëñ x Γëñ left+width`, `top Γëñ y Γëñ top+height` (inclusive)

### generate_false_automap
- **Signature:** `void OverheadMapClass::generate_false_automap(short polygon_index)`
- **Purpose:** Create a limited-visibility automap for checkpoint mode by flood-filling from the starting polygon, respecting secret platform boundaries.
- **Inputs:** `polygon_index` ΓÇö starting polygon (checkpoint location)
- **Outputs/Return:** None (overwrites global automap bitfields; saves originals in `saved_automap_*`)
- **Side effects:**
  - Allocates and saves original automap state to `saved_automap_lines` and `saved_automap_polygons`
  - Clears and rebuilds global automap bitfields
  - Calls `flood_map()` with `false_automap_cost_proc()` cost function (breadth-first)
- **Calls:**
  - `add_poly_to_false_automap()` ΓÇö add starting polygon and its lines to visible set
  - `flood_map()` ΓÇö breadth-first flood-fill with cost function; called first to expand from start, then in a loop with `source_polygon_index=NONE` to process remaining queue
  - Memory allocation/management: `new[]`, `delete[]`, `memcpy()`, `memset()`
- **Notes:**
  - Buffer sizes computed via bit-packing: `(count / 8 + ((count % 8) ? 1 : 0)) * sizeof(byte)`
  - Flood-fill cost function prevents leaving/entering secret platforms, ensuring visibility doesn't penetrate secret doors

### replace_real_automap
- **Signature:** `void OverheadMapClass::replace_real_automap(void)`
- **Purpose:** Restore original automap bitfields after checkpoint-mode rendering.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:**
  - Restores `automap_lines` and `automap_polygons` from saved buffers
  - Deallocates saved buffers and nullifies pointers
- **Calls:** `memcpy()`, `delete[]`
- **Notes:**
  - Defensive: checks if pointers are non-null before copying; handles case where `generate_false_automap()` was not called

### false_automap_cost_proc
- **Signature:** `int32 OverheadMapClass::false_automap_cost_proc(short source_polygon_index, short line_index, short destination_polygon_index, void *caller_data)`
- **Purpose:** Cost function for flood-fill algorithm; returns negative cost to block visibility through secret platforms.
- **Inputs:**
  - `source_polygon_index` ΓÇö polygon being left
  - `line_index` ΓÇö line between source and destination (unused)
  - `destination_polygon_index` ΓÇö polygon being entered
  - `caller_data` ΓÇö unused
- **Outputs/Return:** `int32` ΓÇö cost (1 = passable, -1 = blocked)
- **Side effects:**
  - Calls `add_poly_to_false_automap()` if cost > 0, adding destination to visible set
- **Calls:**
  - `get_polygon_data()` ΓÇö retrieve source and destination polygons
  - `get_platform_data()` ΓÇö check platform type/flags for secret/door status
  - `add_poly_to_false_automap()` ΓÇö add visible destination polygon
- **Notes:**
  - Blocks if source is a secret platform (cost = -1)
  - Blocks if destination is a secret platform that is also a door
  - Destination polygons are added to visible set before returning cost, so the caller can decide whether to expand further

## Control Flow Notes
**Initialization/Frame Sync:**
- Render loop executed once per frame; called after world update
- `Render()` is the entry point; control is delegated to virtual methods in platform-specific subclasses

**Checkpoint Mode Special Case:**
- If `Control.mode == _rendering_checkpoint_map`: 
  1. `generate_false_automap()` saves real automap and creates limited visibility
  2. Normal rendering loop executes with limited polygon/line visibility
  3. Only the goal checkpoint marker is drawn (from `saved_objects`)
  4. Live entities (players, monsters) are skipped
  5. `replace_real_automap()` restores original state
- Otherwise (`_rendering_game_map`): full automap visibility, all entities drawn

**Rendering Order:**
1. Overall setup (`begin_overall()`)
2. Polygons (solid colored regions)
3. Lines (edges between polygons)
4. Annotations (text labels)
5. Paths (if enabled)
6. Entities (players, monsters, items, projectiles)
7. Level name
8. Overall cleanup (`end_overall()`)

## External Dependencies
- **Includes/Headers:**
  - `cseries.h` ΓÇö cross-platform base types, macros
  - `OverheadMapRenderer.h` ΓÇö class definition, color/shape/font configuration data structures
  - `flood_map.h` ΓÇö `flood_map()` algorithm and cost function typedef
  - `media.h` ΓÇö media type enums (`_media_water`, etc.)
  - `platforms.h` ΓÇö platform accessors and flag macros
  - `player.h` ΓÇö player data accessors, team color constants
  - `render.h` ΓÇö rendering flag macros, constants
  - `<string.h>`, `<stdlib.h>`, `<limits.h>` ΓÇö standard C library

- **Game-World Symbols (defined elsewhere):**
  - `dynamic_world` ΓÇö current world state (polygons, lines, endpoints, objects, tick count)
  - `static_world` ΓÇö static level data (level name)
  - `objects`, `saved_objects` ΓÇö entity and saved-object arrays
  - `automap_lines`, `automap_polygons` ΓÇö automap visibility bitfields (global)
  - Accessor functions: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `get_media_data()`, `get_platform_data()`, `get_monster_data()`, `get_player_data()`, `get_next_map_annotation()`, `get_number_of_paths()`, `path_peek()`, `find_flooding_polygon()`, `monster_index_to_player_index()`, `GET_GAME_OPTIONS()`, various flag test/set macros

- **Virtual Methods (implemented by subclasses):**
  - `begin_overall()`, `end_overall()` ΓÇö rendering lifecycle
  - `begin_polygons()`, `end_polygons()`, `draw_polygon()`
  - `begin_lines()`, `end_lines()`, `draw_line()`
  - `begin_things()`, `end_things()`, `draw_thing()`
  - `draw_player()`, `draw_text()`, `draw_annotation()`, `draw_map_name()`
  - `set_path_drawing()`, `draw_path()`, `finish_path()`
