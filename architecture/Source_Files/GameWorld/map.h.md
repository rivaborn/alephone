# Source_Files/GameWorld/map.h

## File Purpose
Core header file for the game world and map system in Aleph One (Marathon engine). Defines all map geometry (vertices, lines, polygons, surfaces), game entities (objects, monsters, projectiles), game state structures, and constants for game mechanics. Provides the central interface for world initialization, entity management, and level progression.

## Core Responsibilities
- Define map geometry primitives (endpoints, lines, sides, polygons) and their relationship graph
- Define game entity types (objects, monsters, items, projectiles, effects) and properties
- Define game state (static level properties, dynamic runtime state, game rules)
- Define game mechanics constants (damage types, transfer modes, difficulty levels, game modes)
- Declare functions for map construction, entity lifecycle, spatial queries, and geometry manipulation
- Manage global world state pointers and entity lists (vectors)
- Provide accessor functions and macros for flag-based property encoding

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `endpoint_data` | struct | 2D map vertex with elevation info; supports variable height polygons |
| `line_data` | struct | Line segment connecting two endpoints; references two polygons (clockwise/counterclockwise) |
| `side_data` | struct | Textured surface attached to a line; includes primary/secondary/transparent textures, control panel data |
| `polygon_data` | struct | Enclosed area with floor/ceiling; stores height, texture, objects, sound sources, neighbors |
| `object_data` | struct | Active world object (monster, item, projectile, effect); 32 bytes, stores animation state |
| `damage_definition` | struct | Damage type, flags, base/random values, difficulty scaling |
| `map_object` (saved_object) | struct | Initial map placement data for objects; read from level file |
| `player_start_data` | struct | Player spawn location with team, identifier, color, name |
| `ambient_sound_image_data` | struct | Non-directional ambient sound; 16 bytes |
| `random_sound_image_data` | struct | Directional/periodic random sound; 32 bytes |
| `static_data` (saved_map_data) | struct | Static level properties: environment, physics, music, mission flags, entry points |
| `dynamic_data` | struct | Runtime level state: tick count, entity counts, random state, player info, statistics |
| `game_data` | struct | Game rules: game type, options, difficulty, kill limit, random seed |
| `object_location` | struct | 3D position and orientation with polygon reference |
| `saved_lighting_function_specification` | struct | Light animation parameters (function, period, intensity) |
| `saved_static_light_data` | struct | Static light definition with 6 animation phases |
| `shape_and_transfer_mode` | struct | Rendering data: collection, shape index, transfer mode, animation frame info |
| Enums: `_damage_*`, `_polygon_*`, `_xfer_*`, `_game_of_*`, `_environment_*` | enum | Game mechanics constants (19 damage types, 23 polygon types, 28 transfer modes, 9 game types, environment flags) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `static_world` | `static_data*` | global | Pointer to static level data (mission flags, environment, music) |
| `dynamic_world` | `dynamic_data*` | global | Pointer to runtime level state (tick count, entity counts, game stats) |
| `ObjectList` | `vector<object_data>` | global | All active game objects (monsters, items, projectiles, effects) |
| `EndpointList` | `vector<endpoint_data>` | global | All map vertices |
| `LineList` | `vector<line_data>` | global | All map line segments |
| `SideList` | `vector<side_data>` | global | All textured surfaces |
| `PolygonList` | `vector<polygon_data>` | global | All enclosed areas |
| `AmbientSoundImageList` | `vector<ambient_sound_image_data>` | global | Ambient sound definitions |
| `RandomSoundImageList` | `vector<random_sound_image_data>` | global | Random/directional sound definitions |
| `MapIndexList` | `vector<int16>` | global | Spatial index for fast polygon/object lookup |
| `AutomapLineList` | `vector<uint8>` | global | Bitmap of visible automap lines |
| `AutomapPolygonList` | `vector<uint8>` | global | Bitmap of visible automap polygons |
| `MapAnnotationList` | `vector<map_annotation>` | global | Text annotations on automap |
| `SavedObjectList` | `vector<map_object>` | global | Initial object placements from map file |
| `game_is_networked` | bool | global | True if this is a network game |
| `LandscapesLoaded` | bool | global | Whether landscape textures have been loaded |
| `LoadedWallTexture` | short | global | Index of main wall texture (for infravision fog) |

## Key Functions / Methods

### update_world
- Signature: `std::pair<bool, int16> update_world(void)`
- Purpose: Main game loop update; advances one tick of simulation
- Inputs: None (reads from `dynamic_world`)
- Outputs/Return: `{changed, elapsed_time}` ΓÇö whether state changed, and time elapsed in ticks
- Side effects: Updates all entity positions, animations, sounds, light states; modifies global game state
- Calls: Monster/projectile update routines (external), `changed_polygon()`, sound/light update functions
- Notes: May perform prediction ahead; returns pair to signal rendering needed even with no elapsed ticks

### initialize_map_for_new_level
- Signature: `void initialize_map_for_new_level(void)`
- Purpose: Per-level setup after map load; resets dynamic state, spawns initial objects
- Inputs: None (uses loaded map data in `PolygonList`, etc.)
- Outputs/Return: None
- Side effects: Initializes `dynamic_world`, calls `place_initial_objects()`, resets player counts, entity counts
- Calls: `place_initial_objects()`, `initialize_control_panels_for_level()`, `update_lightsources()`

### world_point_to_polygon_index
- Signature: `short world_point_to_polygon_index(world_point2d *location)`
- Purpose: Spatial query; find the polygon containing a 2D point
- Inputs: 2D world coordinate
- Outputs/Return: Polygon index, or `NONE` if not in any polygon
- Side effects: None
- Calls: Uses spatial index `MapIndexList` for fast lookup

### point_in_polygon
- Signature: `bool point_in_polygon(short polygon_index, world_point2d *p)`
- Purpose: Test whether a 2D point is inside a specific polygon
- Inputs: Polygon index, 2D point
- Outputs/Return: Boolean containment result
- Side effects: None
- Notes: Used for detailed containment checks after coarse spatial query

### new_map_object / new_map_object2d / new_map_object3d
- Signature: `short new_map_object(object_location*, shape_descriptor); short new_map_object2d(world_point2d*, short, shape_descriptor, angle); short new_map_object3d(world_point3d*, short, shape_descriptor, angle)`
- Purpose: Allocate and insert a new object into the world
- Inputs: Location (2D or 3D), polygon index, shape descriptor, facing angle
- Outputs/Return: Object index (short), or `NONE` if allocation fails
- Side effects: Adds object to `ObjectList`, links into polygon's object list
- Calls: `new_map_object()` (internal wrapper)

### translate_map_object
- Signature: `bool translate_map_object(short object_index, world_point3d *new_location, short new_polygon_index)`
- Purpose: Move an object to a new location and/or polygon
- Inputs: Object index, new 3D location, new polygon index
- Outputs/Return: Boolean success
- Side effects: Updates object location and polygon linkage; may trigger `changed_polygon()` callbacks
- Notes: Validates new polygon and may adjust location to keep object in valid polygon

### remove_map_object
- Signature: `void remove_map_object(short index)`
- Purpose: Remove and deallocate an object
- Inputs: Object index
- Outputs/Return: None
- Side effects: Removes from `ObjectList`, unlinks from polygon object list, marks slot as free

### change_polygon_height
- Signature: `bool change_polygon_height(short polygon_index, world_distance new_floor_height, world_distance new_ceiling_height, damage_definition *damage)`
- Purpose: Move floor or ceiling (platform), optionally applying damage to contained entities
- Inputs: Polygon index, new floor/ceiling heights, optional damage
- Outputs/Return: Boolean success
- Side effects: Updates polygon floor/ceiling, crushes/damages entities inside, triggers neighbor recalculation
- Notes: Handles crushing damage; calls `cause_polygon_damage()`

### calculate_damage
- Signature: `short calculate_damage(damage_definition *damage)`
- Purpose: Compute final damage value from base + random with difficulty scaling
- Inputs: Damage definition (type, flags, base, random, scale)
- Outputs/Return: Final damage amount (int16)
- Side effects: May call random number generator
- Notes: Applies `_alien_damage` scaling based on difficulty level

### cause_polygon_damage
- Signature: `void cause_polygon_damage(short polygon_index, short monster_index)`
- Purpose: Apply environmental damage (lava, suffocation, crushing) to entities in polygon
- Inputs: Polygon index, causing monster index (or `NONE`)
- Outputs/Return: None
- Side effects: Calls `calculate_damage()` and applies damage to all entities in polygon
- Notes: Used for hazard polygons (lava, goo, crushers)

### animate_object
- Signature: `void animate_object(short object_index); void animate_object(object_data* data, int16_t object_index)`
- Purpose: Advance object animation frame by one tick
- Inputs: Object index (or pointer + index)
- Outputs/Return: None
- Side effects: Updates `object_data.sequence` field (frame and phase); sets animation flags
- Notes: Sets `_obj_animated` flags to signal renderer; `phase` field tracks animation timing

### get_object_shape_and_transfer_mode
- Signature: `void get_object_shape_and_transfer_mode(world_point3d *camera_location, short object_index, shape_and_transfer_mode *data); void get_object_shape_and_transfer_mode(world_point3d *camera_location, object_data* object, shape_and_transfer_mode *data)`
- Purpose: Retrieve rendering data (shape, collection, transfer mode, animation frame)
- Inputs: Camera location (for LOD), object index or pointer
- Outputs/Return: Fills `shape_and_transfer_mode` struct with rendering parameters
- Side effects: May perform LOD or animation state queries
- Notes: Adapts shape descriptor and animation frame for renderer; handles both indexed and pointer access

### line_is_obstructed
- Signature: `bool line_is_obstructed(short polygon_index1, world_point2d* p1, short polygon_index2, world_point2d* p2, bool for_sounds)`
- Purpose: Test whether a line segment is blocked by walls or other geometry
- Inputs: Two polygon indices and 2D points, optional sound flag
- Outputs/Return: Boolean obstruction result
- Side effects: None
- Calls: Traces line through polygon graph
- Notes: Used for line-of-sight, projectile deflection, and sound propagation

### changed_polygon
- Signature: `void changed_polygon(short original_polygon_index, short new_polygon_index, short player_index)`
- Purpose: Callback when entity changes polygons; activates triggers and updates light/sound context
- Inputs: Original and new polygon indices, player index
- Outputs/Return: None
- Side effects: Activates light/platform/sound triggers; updates player environment; calls `entered_polygon()`, `left_polygon()`
- Notes: Handles entry/exit logic for teleporters, hazards, goal areas

### place_initial_objects
- Signature: `void place_initial_objects(void)`
- Purpose: Spawn all objects defined in `SavedObjectList` (from map file)
- Inputs: None (reads `SavedObjectList`)
- Outputs/Return: None
- Side effects: Creates monsters, items, scenery from map data; calls `new_map_object()` for each
- Notes: Called during `initialize_map_for_new_level()`

## Control Flow Notes

**Initialization Phase:**
- `initialize_marathon()` ΓåÆ `new_game()` ΓåÆ `generate_map()` (load level from file) ΓåÆ `initialize_map_for_new_level()` ΓåÆ `place_initial_objects()`

**Main Game Loop (per-frame):**
- `update_world()` ticks all systems (monsters, projectiles, platforms, lights, sounds)
- Returns pair indicating whether rendering is needed and elapsed time
- Calls `changed_polygon()` when entities change rooms, triggering hazards/triggers

**Shutdown:**
- `leaving_map()` cleans up level state before loading next level

The file is geometry-centric; external code in other modules manages monsters (`monster.c`), projectiles, and effects. This file provides the spatial substrate and global state management.

## External Dependencies
- **csmacros.h** ΓÇö Macro utilities: `TEST_FLAG`, `SET_FLAG`, `MAX`, `MIN`, pinning/clamping macros
- **world.h** ΓÇö World coordinate types (`world_point2d`, `world_point3d`, `world_distance`, `angle`, `_fixed`), trigonometry, distance functions
- **dynamic_limits.h** ΓÇö Dynamic limit accessor `get_dynamic_limit()` for entity caps
- **shape_descriptors.h** ΓÇö Shape descriptor type and macros (`BUILD_DESCRIPTOR`, `GET_DESCRIPTOR_SHAPE`, etc.)
- **`<vector>`** ΓÇö STL vector for dynamic arrays (ObjectList, PolygonList, etc.)
- External symbols (defined elsewhere): Monster/projectile managers, rendering system, audio system, physics engine
