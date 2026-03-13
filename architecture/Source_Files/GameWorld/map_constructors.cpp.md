# Source_Files/GameWorld/map_constructors.cpp

## File Purpose
Handles construction and precalculation of map geometry for the Marathon engine. Computes derived properties of polygons, lines, sides, and endpoints (vertices) to optimize runtime queries. Provides serialization/deserialization (pack/unpack) functions for all major map data structures.

## Core Responsibilities
- **Side type calculation**: Determines visual side types (_full_side, _high_side, _low_side, _split_side) based on height relationships between adjacent polygons.
- **Redundant data recalculation**: Computes cached properties (area, center, adjacent geometry, solidity flags) for polygons, endpoints, lines, and sides.
- **Map index precalculation**: Builds exclusion zones and neighbor lists using breadth-first flood-fill.
- **Geometry ownership tracking**: Determines which polygons and lines own each endpoint.
- **Lighting assignment**: Guesses appropriate light source indices for sides based on polygon types and side categories.
- **Sound source discovery**: Identifies which map-placed sound sources are near each polygon.
- **Serialization/deserialization**: Pack/unpack functions for all primary map data types (endpoint, line, side, polygon, map objects, etc.) to/from byte streams.

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `intersecting_flood_data` | struct | Temporary state during flood-fill traversal; tracks center point and minimum separation distance. |
| `endpoint_data` | struct (from map.h) | Vertex with height/solidity/elevation flags and supporting polygon reference. |
| `line_data` | struct (from map.h) | Edge connecting two endpoints; owns clockwise/counterclockwise sides and adjacent polygon references. |
| `side_data` | struct (from map.h) | Visible face of a line as seen from a polygon; includes textures, exclusion zone, control panel data, light sources. |
| `polygon_data` | struct (from map.h) | Enclosed area (room/sector); owns vertices, lines, sides, adjacent polygons, exclusion zones, neighbor list, sound sources. |
| `platform_data` | struct (from platforms.h) | Moving platform with static/dynamic flags, height bounds, parent reference. |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `map_index_buffer_count` | `int32` | static | Size of the map index buffer in shorts; set via `set_map_index_buffer_size()`. |
| `LineIndices` | `vector<short>` | static | Temporary list of line indices found during flood-fill; reused across calls. |
| `EndpointIndices` | `vector<short>` | static | Temporary list of endpoint indices found during flood-fill; reused across calls. |
| `PolygonIndices` | `vector<short>` | static | Temporary list of polygon indices found during flood-fill; reused across calls. |
| `MapIndexList` | `vector<int16>` | global (extern) | Master list of all map indices (used by exclusion zones and neighbors); size tracked in `dynamic_world->map_index_count`. |

## Key Functions / Methods

### recalculate_side_type
- **Signature**: `void recalculate_side_type(short side_index)`
- **Purpose**: Determine and assign the correct side type based on height relationships with the adjacent polygon.
- **Inputs**: Side index.
- **Outputs/Return**: None (modifies `side->type` in place).
- **Side effects**: Reads adjacent polygon heights (and platform data if adjacent polygon is a platform); updates side type field.
- **Calls**: `get_side_data()`, `find_adjacent_polygon()`, `get_polygon_data()`, `get_platform_data()`.
- **Notes**: Handles platforms by querying their min/max heights. If no adjacent polygon (exterior edge), always sets `_full_side`.

### new_side
- **Signature**: `short new_side(short polygon_index, short line_index)`
- **Purpose**: Create a new side owned by the given polygon on the given line, add it to `SideList`, and link it to the line.
- **Inputs**: Polygon index, line index.
- **Outputs/Return**: New side index.
- **Side effects**: Appends to `SideList`, increments `dynamic_world->side_count`, updates line's side index pointers, calls `recalculate_redundant_side_data()` and `recalculate_side_type()`.
- **Calls**: `get_line_data()`, `get_polygon_data()`, `recalculate_redundant_side_data()`, `calculate_adjacent_sides()`, `recalculate_side_type()`.
- **Notes**: Asserts that the side does not already exist on the line for this polygon.

### recalculate_redundant_polygon_data
- **Signature**: `void recalculate_redundant_polygon_data(short polygon_index)`
- **Purpose**: Recompute cached polygon properties: clockwise endpoint order, adjacent polygons, area, center, and side references.
- **Inputs**: Polygon index.
- **Outputs/Return**: None (modifies polygon fields in place).
- **Side effects**: Calls `calculate_clockwise_endpoints()`, `calculate_adjacent_polygons()`, `calculate_polygon_area()`, `find_center_of_polygon()`, `calculate_adjacent_sides()`.
- **Calls**: Functions listed above; skips all calculations if polygon is detached.
- **Notes**: Does not compute exclusion zones or neighbor lists; see `precalculate_map_indexes()` for that. Skips calculations if `POLYGON_IS_DETACHED(polygon)` is true.

### precalculate_map_indexes
- **Signature**: `void precalculate_map_indexes(void)`
- **Purpose**: Main entry point; precalculates exclusion zones (lines/endpoints within MINIMUM_SEPARATION_FROM_WALL) and neighbor lists (polygons within MINIMUM_SEPARATION_FROM_PROJECTILE) for all non-detached polygons.
- **Inputs**: None (operates on global polygon list).
- **Outputs/Return**: None (populates `MapIndexList` and sets polygon index fields).
- **Side effects**: For each polygon: calls `find_intersecting_endpoints_and_lines()` twice (different separation distances), appends results to `MapIndexList`, updates polygon's `first_exclusion_zone_index`, `line_exclusion_zone_count`, `point_exclusion_zone_count`, `first_neighbor_index`, `neighbor_count`. Calls `precalculate_polygon_sound_sources()` at end.
- **Calls**: `find_intersecting_endpoints_and_lines()`, `add_map_index()`, `precalculate_polygon_sound_sources()`.
- **Notes**: Runs after map loading to enable collision detection and neighbor queries at runtime.

### intersecting_flood_proc
- **Signature**: `static int32 intersecting_flood_proc(short source_polygon_index, short line_index, short destination_polygon_index, void *vdata)`
- **Purpose**: Callback for breadth-first flood-fill; checks if the source polygon overlaps the original polygon's Z-range and if its lines/endpoints are within the minimum separation distance; populates global `LineIndices`, `EndpointIndices`, `PolygonIndices` vectors.
- **Inputs**: Source polygon, crossing line, destination polygon (unused), caller data (cast to `intersecting_flood_data*`).
- **Outputs/Return**: 1 to continue flooding (if something was found close enough), ΓêÆ1 to stop.
- **Side effects**: Appends to `LineIndices`, `EndpointIndices`, `PolygonIndices` if candidates are found.
- **Calls**: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `LINE_IS_SOLID()`, `line_has_variable_height()`, `point_to_line_segment_distance_squared()`.
- **Notes**: Performs Z-range overlap check first; only processes lines/endpoints if Z-ranges intersect. Skips duplicates in the result vectors. Uses squared distances to avoid sqrt().

### guess_side_lightsource_indexes
- **Signature**: `void guess_side_lightsource_indexes(short side_index)`
- **Purpose**: Assign light source indices to a side's primary, secondary, and transparent texture based on side type and polygon lighting.
- **Inputs**: Side index.
- **Outputs/Return**: None (modifies side light source fields).
- **Side effects**: Reads polygon and line height data; may query adjacent flooded platform. Updates `side->primary_lightsource_index`, `side->secondary_lightsource_index`, `side->transparent_lightsource_index`.
- **Calls**: `get_side_data()`, `get_line_data()`, `get_polygon_data()`, `get_platform_data()`, `find_flooding_polygon()`.
- **Notes**: Handles M1 net map orphan sides (bounds check). For platforms, may override floor light if platform is flooded. Switch-based logic selects light index based on side type.

### unpack_* / pack_* (serialization functions)
- **Signatures**: `uint8 *unpack_<type>(uint8 *Stream, <type> *Objects, size_t Count)` and `uint8 *pack_<type>(uint8 *Stream, <type> *Objects, size_t Count)` for types: endpoint_data, line_data, side_data, polygon_data, map_annotation, map_object, object_frequency_definition, static_data, ambient_sound_image_data, random_sound_image_data, dynamic_data, object_data, damage_definition.
- **Purpose**: Convert between packed big-endian byte stream and native memory layout (with alignment).
- **Inputs**: Byte stream pointer, object array, count.
- **Outputs/Return**: Advanced byte stream pointer.
- **Side effects**: Reads from or writes to stream; updates stream pointer. Uses `StreamToValue()`, `ValueToStream()`, `StreamToList()`, `ListToStream()`, `StreamToBytes()`, `BytesToStream()` macros from Packing.h.
- **Notes**: Each function asserts that bytes consumed equals `Count * SIZEOF_<type>`. M1 compatibility: `unpack_map_object()` has version-specific logic. No explicit error handling; asserts on mismatch.

## Control Flow Notes
**Initialization sequence** (inferred):
1. Map file loaded ΓåÆ raw byte streams unpacked into structures via unpack_* functions.
2. `recalculate_redundant_polygon_data()` called for each polygon to compute clockwise endpoints, area, center, adjacent polygons, and sides.
3. `precalculate_map_indexes()` called once to build exclusion zones and neighbor lists via flood-fill.
4. Collision detection and neighbor queries use the precalculated maps at runtime.

**Runtime**:
- `recalculate_side_type()` called when geometry changes (e.g., platform height changes).
- `guess_side_lightsource_indexes()` may be called when lights are added/removed or when editing the map.
- Pack/unpack functions used during save/load and network transmission.

## External Dependencies
- **Includes**: `cseries.h` (base types, SDL), `editor.h` (data version constants), `map.h` (primary geometry types, global lists), `flood_map.h` (flood-fill function), `platforms.h` (platform types/flags), `Packing.h` (serialization macros).
- **Defined elsewhere**:
  - `map_lines`, `map_polygons`, `map_sides`, `map_endpoints` (arrays from map.h globals).
  - `SideList`, `EndpointList`, `LineList`, `PolygonList`, `MapIndexList` (vector globals from map.h).
  - `dynamic_world` (global dynamic_data pointer).
  - `get_polygon_data()`, `get_line_data()`, `get_side_data()`, `get_endpoint_data()` (accessor functions).
  - `flood_map()` (breadth-first flood-fill from flood_map.h).
  - `find_center_of_polygon()`, `point_to_line_segment_distance_squared()`, `push_out_line()`, `find_adjacent_polygon()`, `find_flooding_polygon()`, `line_has_variable_height()` (geometry utility functions).
  - `film_profile` (global with feature flags, including `adjacent_polygons_always_intersect`).
