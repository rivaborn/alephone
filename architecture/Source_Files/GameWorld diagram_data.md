# Source_Files/GameWorld/devices.cpp
## File Purpose
Manages interactive control panels in the game worldΓÇöswitches, terminals, refuel stations, and save points. Handles panel initialization, player/projectile interaction, state synchronization with linked platforms and lights, and MML configuration support.

## Core Responsibilities
- Initialize control panel states based on linked platforms/lights at level start
- Update panel states during gameplay (refueling, save game cooldown)
- Detect player action-key targets (panels, platforms) within activation range
- Toggle panels via player action or projectile hit
- Synchronize panel visual state and associated sounds
- Handle specialized panels: terminals, pattern buffers (save points), refuel stations
- Parse and reset MML-based configuration (activation ranges, energy amounts, sounds)
- Invoke Lua script hooks on panel state changes

## External Dependencies

- **map.h** ΓÇô `side_data`, `polygon_data`, `line_data`, `get_*_data()`, `dynamic_world`, map globals
- **monsters.h** ΓÇô `get_monster_data()`, `get_monster_dimensions()`, `monster_index`
- **player.h** ΓÇô `player_data`, `players[]`, player state and item management
- **platforms.h** ΓÇô `get_polygon_data()`, `platform_is_on()`, `try_and_change_platform_state()`, platform state
- **SoundManager.h** ΓÇô `SoundManager::instance()->StopSound()`
- **computer_interface.h** ΓÇô `enter_computer_interface()`, terminal mode
- **lightsource.h** ΓÇô `get_light_status()`, `set_light_status()`, `get_light_intensity()`
- **lua_script.h** ΓÇô `L_Call_*()` hooks (Lua callbacks)
- **InfoTree.h** ΓÇô MML XML parsing


# Source_Files/GameWorld/dynamic_limits.cpp
## File Purpose
Manages runtime-configurable limits for game world entity pools (objects, monsters, effects, projectiles, etc.). Provides three preset configurations (Marathon 2 original, Aleph One 1.0 expanded, and 1.1 compatible) and allows override via MML XML markup. Coordinates reallocation of backing arrays in dependent subsystems when limits change.

## Core Responsibilities
- Maintain three static limit configurations for different game versions/compatibility modes
- Parse MML XML configuration to override individual entity limits
- Select and initialize appropriate limit preset based on film profile flags
- Reallocate entity storage arrays across subsystems when limits change
- Provide accessor function for runtime limit queries
- Ensure consistency between limit values and allocated array sizes

## External Dependencies
- **Includes (defined elsewhere):**
  - `cseries.h` ΓÇö core series utilities and types
  - `map.h` ΓÇö map structures; defines `MAXIMUM_*_PER_MAP` macros that call `get_dynamic_limit()`
  - `effects.h`, `monsters.h`, `projectiles.h` ΓÇö entity subsystem headers
  - `ephemera.h` ΓÇö render effect particles
  - `flood_map.h` ΓÇö pathfinding support
  - `InfoTree.h` ΓÇö XML tree parser for MML
- **Global symbols used (defined elsewhere):**
  - `EffectList`, `ObjectList`, `MonsterList`, `ProjectileList` ΓÇö std::vector entity pools (from effects.cpp, map.cpp, monsters.cpp, projectiles.cpp)
  - `film_profile` ΓÇö global object tracking Marathon version compatibility flags
  - `allocate_pathfinding_memory()`, `allocate_ephemera_storage()` ΓÇö subsystem allocation routines

# Source_Files/GameWorld/dynamic_limits.h
## File Purpose
Header file defining the interface for querying and configuring dynamic resource limits in the game engine. Supports limits on objects, NPCs, projectiles, effects, and collision structuresΓÇöall initialized from XML configuration and retrievable at runtime.

## Core Responsibilities
- Define enumeration of configurable entity limit types (objects, monsters, projectiles, effects, collision buffers, ephemera, garbage)
- Declare XML parser to load limit values from configuration
- Provide accessor function to retrieve current limit by type
- Support reset of limits to initial defaults

## External Dependencies
- `#include "cstypes.h"` ΓÇô standard integer types (`uint16`, `int`)
- Forward declaration: `class InfoTree` ΓÇô XML/config tree structure (defined elsewhere; likely in a config/parsing module)

# Source_Files/GameWorld/editor.h
## File Purpose
Header file defining version constants for the Aleph One game engine's map editor and data format compatibility. Tracks Marathon engine versions (One, Two, Infinity) to manage file format evolution.

## Core Responsibilities
- Define editor map data format version constants
- Provide version identifiers for Marathon engine iterations (1.0, 2.0, Infinity)
- Establish current editor map version by aliasing to latest supported format

## External Dependencies
- Standard C header guard pattern (`__EDITOR_H_`)
- No external includes or dependencies
- Designed to be included by other game world editor modules that need version information

**Notes:**
- Version constant scheme (0, 1, 2) allows serialization and compatibility checks when loading/saving map files across different Marathon engine versions.
- Comment indicates `EDITOR_MAP_VERSION` was updated in Feb 2000 to support Infinity format.

# Source_Files/GameWorld/effect_definitions.h
## File Purpose
Defines static configuration data for visual and audio effects that occur in the game world (explosions, blood splashes, teleportation, water/lava/sewage splashes, etc.). Maps each effect type to sprite animation properties and sound parameters.

## Core Responsibilities
- Define effect property struct and behavior flags
- Declare static effect definition database
- Store animation collection/shape mappings for all ~67 effect types
- Provide serialization functions for save/load operations
- Support effect variants for different media types (water, lava, sewage, goo, Jjaro)

## External Dependencies
- **effects.h**: Defines effect type enum (e.g., `_effect_rocket_explosion`, `_effect_teleport_object_in`), `NUMBER_OF_EFFECT_TYPES` constant, and prototypes for `new_effect()`, `update_effects()`, `remove_effect()`, serialization functions
- **map.h**: Provides constants like `TICKS_PER_SECOND`, `NONE` macro, and serialization infrastructure
- **SoundManagerEnums.h**: Provides sound index constants (e.g., `_snd_teleport_in`) and frequency constants (`_normal_frequency`, `_higher_frequency`, `_lower_frequency`)

# Source_Files/GameWorld/effects.cpp
## File Purpose
Manages visual and audio effects in the game world (explosions, blood splashes, teleportation, media interactions, etc.). Effects are time-limited visual/audio objects that animate and clean themselves up based on definition flags and animation completion.

## Core Responsibilities
- Create and destroy effect instances at runtime
- Update effect animations and state each frame tick
- Handle special effects logic (teleportation object visibility toggling, delay handling)
- Manage effect collection loading/unloading
- Serialize and deserialize effect state for save games
- Integrate with the Lua scripting system for effect invalidation

## External Dependencies
- **map.h**: World geometry, object structures (`object_data`, `polygon_data`), map object functions (`new_map_object3d()`, `remove_map_object()`)
- **interface.h**: Shape/collection info (`get_shape_animation_data()`, `mark_collection_for_loading()`)
- **SoundManager.h**: Sound playback (`play_world_sound()`, `play_object_sound()`, `Sound_TeleportOut()`)
- **lua_script.h**: Lua integration (`L_Invalidate_Effect()`)
- **Packing.h**: Binary I/O utilities (`StreamToValue()`, `ValueToStream()`)
- **effect_definitions.h**: Effect template data and flags (included for definitions)
- **world.h**: 3D geometry types (`world_point3d`, `angle`)

# Source_Files/GameWorld/effects.h
## File Purpose
Defines the interface for managing in-world visual effects (explosions, blood splashes, projectile contrails, teleportation effects, etc.). Provides effect creation, update, and lifecycle management with dynamic limits and serialization support for saving/loading game state.

## Core Responsibilities
- Define 70+ effect types (weapons, blood, environment interactions, creature-specific)
- Manage active effects via dynamic vector storage with runtime-adjustable limits
- Provide effect instance lifecycle: creation, per-frame updates, removal
- Handle effect persistence flags and activation delays
- Serialize/deserialize effect data and definitions for save files and M1 compatibility

## External Dependencies
- **Includes:**
  - `world.h` ΓÇô world coordinate types (`world_point3d`, `angle`) and spatial primitives
  - `dynamic_limits.h` ΓÇô `get_dynamic_limit()` for runtime effect cap
  - `<vector>` ΓÇô C++ standard library dynamic array
- **Defined elsewhere:** Effect definition structures, implementations of all declared functions, resource loading

# Source_Files/GameWorld/ephemera.cpp
## File Purpose
Manages ephemeraΓÇöshort-lived visual objects (effects, particles) in the game world. Implements an object pool allocator for efficient reuse and organizes ephemera spatially by polygon for fast updates and lookups.

## Core Responsibilities
- Object pool allocation/deallocation (linked-list free-list design)
- Create and initialize ephemera with shape, location, and animation state
- Remove ephemera and invalidate Lua references
- Maintain per-polygon linked lists of ephemera for spatial queries
- Animate ephemera each frame and auto-remove when animation loops
- Track shape animation data and transfer modes

## External Dependencies
- **dynamic_limits.h** ΓÇö `get_dynamic_limit(_dynamic_limit_ephemera)` for pool size
- **interface.h** ΓÇö `get_shape_animation_data()` to fetch animation metadata
- **lua_script.h** ΓÇö `L_Invalidate_Ephemera()` to notify Lua of removals
- **map.h** ΓÇö `object_data` structure; `animate_object()`, `local_random()`, macros (`MARK_SLOT_AS_FREE`, `MARK_SLOT_AS_USED`, `BUILD_SEQUENCE`, `NONE`)
- **shapes_file_is_m1()** ΓÇö (defined elsewhere) branching for Marathon 1 vs. 2+ shape behavior

# Source_Files/GameWorld/ephemera.h
## File Purpose
Defines the interface for managing ephemeral objectsΓÇötemporary visual effects and animated sprites that exist in the game world for limited durations. Ephemera are lightweight objects stored in polygons, with automatic cleanup based on animation state.

## Core Responsibilities
- Allocate and initialize ephemera storage pools per level
- Create ephemera at world positions with visual representation (shape descriptor)
- Manage ephemera lifecycle (creation, removal, garbage collection)
- Maintain polygon-based spatial indexing of ephemera
- Update ephemera animation state each frame
- Track which polygons were rendered to optimize cleanup

## External Dependencies
- **map.h**: `object_data` (stores ephemera state), `polygon_data` (spatial structure)
- **shape_descriptors.h**: `shape_descriptor` (visual representation)
- **world.h**: `world_point3d`, `angle` (position and orientation types)

# Source_Files/GameWorld/flood_map.cpp
## File Purpose
Implements a polygon-based graph search (flood fill) algorithm for pathfinding in the game world. Supports multiple search strategies (best-first, breadth-first) to expand polygons incrementally and retrieve paths by backtracking through parent pointers.

## Core Responsibilities
- Allocate and manage search tree memory (nodes and visited polygon tracking)
- Perform incremental graph expansion with configurable cost functions and search modes
- Track the path from source to any reached polygon via parent node pointers
- Support path retrieval via backward walk from destination to source
- Provide utility to select random expanded nodes with optional directional bias

## External Dependencies
- **Includes:** `cseries.h`, `map.h`, `flood_map.h`, standard C++ (`<string.h>`, `<stdlib.h>`, `<limits.h>`).
- **Defined elsewhere:**
  - `polygon_data`, `node_data` (map.h)
  - `get_polygon_data()` ΓÇö accessor for polygon structures.
  - `find_center_of_polygon()` ΓÇö computes polygon center.
  - `global_random()` ΓÇö RNG.
  - `objlist_set()` ΓÇö macro to fill array.
  - `MAXIMUM_POLYGONS_PER_MAP`, `MAXIMUM_FLOOD_NODES` ΓÇö limits.
  - `dynamic_world` ΓÇö global world state.
- **Namespace:** `alephone::flood_map` (internal struct defined here; functions are module-scoped).

# Source_Files/GameWorld/flood_map.h
## File Purpose
Header file declaring the pathfinding and flood-fill system for the game engine. Provides interfaces for computing paths between world polygons, moving entities along paths, and performing flood-fill operations with customizable cost algorithms for AI navigation and reachability analysis.

## Core Responsibilities
- Declare pathfinding memory management and path lifecycle functions
- Define flood-fill algorithm modes and customizable cost calculation interface
- Provide path creation, traversal, and deletion operations
- Declare flood-map computation with multiple algorithm strategies
- Support random sampling from computed flood areas with optional spatial bias
- Enable cost-based path optimization via user-supplied cost functions

## External Dependencies
- **Includes:** `world.h` ΓÇö provides `world_point2d`, `world_distance`, `world_vector2d` types, polygon indices, and world coordinate systems.
- **Defined elsewhere:** Memory allocators, path and flood-map storage, cost calculations, random selection logic.

# Source_Files/GameWorld/interpolated_world.cpp
## File Purpose

Provides high-frame-rate rendering (>30 FPS) by interpolating world state between fixed 30 FPS game ticks. Maintains double-buffered snapshots of game world (objects, polygons, sides, lines, ephemera, camera, weapons) and blends them each render frame based on elapsed time.

## Core Responsibilities

- Initialize and manage double-buffered world state snapshots
- Capture complete world state at each 30 FPS tick boundary
- Calculate per-frame interpolation progress (0ΓÇô1) based on machine tick counter
- Interpolate object positions with speed-based rejection (don't interpolate fast-moving projectiles)
- Interpolate camera rotation and position with angle wraparound handling
- Interpolate polygon/line geometry heights and side texture offsets
- Handle object/ephemera polygon crossings during interpolation
- Track projectile positions for contrail effect rendering
- Interpolate weapon display information (ammo count, shell positions)

## External Dependencies

- **Standard library:** `<cmath>` (std::round), `<cstdint>`, `<vector>`
- **Game world data:** `objects`, `map_polygons`, `map_sides`, `map_lines` (vectors defined in map.h); `polygon_ephemera` (extern); `world_view` (extern)
- **Limits:** `get_dynamic_limit(_dynamic_limit_ephemera)` from dynamic_limits.h
- **Ephemera:** `get_ephemera_data()`, `remove_ephemera_from_polygon()`, `add_ephemera_to_polygon()` from ephemera.h
- **Map queries:** `get_line_data()`, `find_new_object_polygon()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()` from map.h
- **Rendering:** `update_world_view_camera()` (render.h/cpp), `TEST_RENDER_FLAG()` (render.h)
- **Preferences:** `get_fps_target()` from preferences.h
- **Timing:** `machine_tick_count()` (defined elsewhere), `TICKS_PER_SECOND`, `MACHINE_TICKS_PER_SECOND` (map.h constants)
- **Movie:** `Movie::instance()->IsRecording()` from Movie.h
- **Replay:** `game_is_being_replayed()`, `get_replay_speed()` (defined elsewhere)
- **Weapons:** `get_weapon_display_information()`, `weapon_display_information` type from weapons.h

# Source_Files/GameWorld/interpolated_world.h
## File Purpose
Declares the public interface for a high-frequency world interpolation system that enables smooth frame rates above 30 FPS by interpolating game state and camera view between fixed update ticks. Part of the Aleph One game engine.

## Core Responsibilities
- Lifecycle management (init, enter, exit) of interpolated world context
- Per-frame state interpolation driven by heartbeat fraction
- Camera/view interpolation for smooth visual movement
- Contrail effect tracking during interpolation
- Weapon display data retrieval for rendering

## External Dependencies
- `<cstdint>` ΓÇô `int16_t` type
- `weapon_display_information` struct ΓÇô defined elsewhere (likely in HUD/weapon rendering code)

# Source_Files/GameWorld/item_definitions.h
## File Purpose
Defines the static item metadata table for the game engine. Contains definitions for all itemsΓÇöweapons, ammunition, powerups, special items (keys, balls), and recharge stationsΓÇöused during inventory management and item spawning.

## Core Responsibilities
- Define the `item_definition` struct to hold item metadata (kind, names, shape, counts)
- Declare static array `item_definitions[]` containing ~50 item entries indexed by item type
- Support per-difficulty item count limits via extended array
- Provide method to query maximum carry count for a given difficulty level
- Prevent duplicate definitions in script-generated code via `DONT_REPEAT_DEFINITIONS` guard

## External Dependencies
- **items.h:** Enum definitions (`_weapon`, `_ammunition`, `_powerup`, `_item`, `_ball`; item type IDs like `_i_knife`, `_i_magnum`, etc.)
- **map.h:** Type definitions (`shape_descriptor`, `BUILD_DESCRIPTOR`, `UNONE`, game difficulty constants)
- **Conditional compilation guard:** `DONT_REPEAT_DEFINITIONS` (used by script_instructions.cpp to avoid duplication)

# Source_Files/GameWorld/items.cpp
## File Purpose
Manages item lifecycle in the game world: creation, placement, pickup mechanics, inventory updates, and animation. Handles both regular items (weapons, ammunition) and special item types (powerups, balls, team-specific items).

## Core Responsibilities
- Create and place new items in the world via `new_item()`
- Handle item pickup logic and inventory updates via `try_and_add_player_item()`
- Manage item visibility and collection (network-only, single-player filtering)
- Animate item shapes each frame via `animate_items()`
- Trigger hidden items to appear in specific zones via `trigger_nearby_items()`
- Support nearby item auto-pickup ("swiping") via `swipe_nearby_items()`
- Parse and manage MML-based item configuration
- Provide item metadata accessors (kind, shape, name)

## External Dependencies
- **map.h**: object data, polygon data, line data, dynamic_world, static_world, map geometry functions
- **player.h**: player data, inventory arrays, item pickup sounds
- **monsters.h**: monster data access for player's monster index
- **weapons.h**: `process_new_item_for_reloading()` (weapon integration)
- **SoundManager.h**: `PlaySound()`, sound index accessors (pickup sounds, powerup sounds)
- **fades.h**: `start_fade()` (screen flash on pickup)
- **interface.h**: collection marking, shape animation data
- **item_definitions.h**: `item_definitions[]` global array, `item_definition` struct
- **flood_map.h**: `flood_map()` (polygon reachability)
- **lua_script.h**: `L_Call_Item_Created()`, `L_Call_Got_Item()` (Lua hooks)
- **InfoTree.h**: XML parsing for MML configuration
- **FilmProfile.h**: `film_profile.swipe_nearby_items_fix` (compatibility flag)
- **network_games.h**: game type/option checks (CTF, Rugby, multiplayer)

# Source_Files/GameWorld/items.h
## File Purpose
Header file defining the item system for the Aleph One game engine (Marathon-based). Declares enumerations for item types and categories, and provides prototypes for item creation, inventory management, and XML configuration parsing.

## Core Responsibilities
- Define item type enumerations (weapon, ammunition, powerup, ball, etc.)
- Declare item lifecycle functions (creation, animation, initialization)
- Declare inventory/player item interaction functions
- Declare item query utilities (shape, validity, kind)
- Declare XML/MML configuration parsing for item customization

## External Dependencies
- `object_location` ΓÇö struct for world coordinates/orientation (defined elsewhere).
- `InfoTree` ΓÇö XML parsing class (MML/XML configuration format).
- References to player and polygon systems (CTF ball, triggers).

**Notes**: This is a pure declaration header; implementation is in `items.c`. The file evolved significantly (Feb 2000ΓÇôMay 2000) to add SMG weapons, XML customization, and animator/initializer patterns borrowed from `scenery.h`.

# Source_Files/GameWorld/lightsource.cpp
## File Purpose
Implements the dynamic lighting system for the Marathon game engine (Aleph One). Manages light creation, state transitions, animation curves, and intensity calculations. Supports six-phase state machines, multiple animation function types (constant, linear, smooth, flicker, random, fluorescent), and legacy Marathon 1 light data conversion.

## Core Responsibilities
- Create and manage dynamic light instances (`new_light()`)
- Animate lights via phased state machines (6 states: becoming/primary/secondary active/inactive)
- Calculate light intensities each frame based on parameterized animation curves
- Handle state transitions when animation periods expire (`rephase_light()`)
- Query and modify light activation status (`get_light_status()`, `set_light_status()`)
- Support tagged light groups for trigger-based activation
- Serialize/deserialize light data for save games and map loading
- Convert Marathon 1 legacy light data to Marathon 2 format

## External Dependencies

- `map.h` ΓÇö Map structures, MAXIMUM_LIGHTS_PER_MAP, slot macros (SLOT_IS_USED, MARK_SLOT_AS_USED), global_random()
- `lightsource.h` ΓÇö Header (light_data, static_light_data, lighting_function_specification definitions, constants, prototypes)
- `Packing.h` ΓÇö StreamToValue(), ValueToStream() for byte serialization
- `lua_script.h` ΓÇö L_Call_Light_Activated() hook
- `cseries.h` ΓÇö Core types (_fixed, uint8), macros (TEST_FLAG16, SET_FLAG16), utilities (csprintf, vhalt)

**Defined elsewhere (not in this file):**
- `cosine_table`, `TRIG_MAGNITUDE`, `TRIG_SHIFT`, `HALF_CIRCLE` (trig tables, math)
- `GetMemberWithBounds()` (bounds-checking accessor helper)
- `assume_correct_switch_position()` (panel/device state update)
- `obj_copy()` (struct assignment helper)

# Source_Files/GameWorld/lightsource.h
## File Purpose
Defines light source data structures, lighting function specifications, and state management for the game engine's dynamic lighting system. Provides interfaces for creating, updating, and querying lights, with backward compatibility for Marathon I map format.

## Core Responsibilities
- Define static light data structures with type and behavior flags
- Specify lighting intensity transition functions (constant, linear, smooth, flicker, random, fluorescent)
- Define dynamic light state tracking (active/inactive states, current intensity, phase)
- Manage global light list as a dynamic array
- Provide light status queries and state mutation operations
- Support serialization/deserialization of light data
- Maintain Marathon I format compatibility layer for legacy map support

## External Dependencies
- `cstypes.h`: Provides `_fixed` (16.16 fixed-point), `int16`, `uint16`, `uint8` types
- `<vector>`: C++ standard library for dynamic light array
- Not inferable: Implementation of lighting functions, state machine logic (defined elsewhere in .c file)

# Source_Files/GameWorld/map.cpp
## File Purpose
Core implementation of the game world map system for the Marathon engine. Manages map initialization, object lifecycle, geometry manipulation, collision detection, sound propagation, and texture resource loading across all game states.

## Core Responsibilities
- Memory allocation and lifecycle management for dynamic map structures
- Object creation, movement, and removal with polygon-spatial organization
- Polygon geometry queries and modification (height changes, collisions)
- Sound propagation and obstruction detection in 3D world
- Texture collection management and environment-based resource loading
- Deferred object operations to support prediction/networking features
- Garbage collection for dead bodies and spent objects

## External Dependencies
- Notable includes: map.h (API), FilmProfile.h (compat flags), interface.h (shapes/collections), monsters.h/projectiles.h/effects.h (entities), SoundManager.h (audio), InfoTree.h (XML parsing)
- Defined elsewhere: GetMemberWithBounds() (bounds helper), obj_clear/objlist_clear (memory), get_dynamic_limit() (resource limits), geometric helpers (closest_point_on_line, find_adjacent_polygon), entity accessors (get_monster_data, get_media_data)
- Lua integration: L_Invalidate_Object() called on deletion for script cleanup

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

## External Dependencies
- **csmacros.h** ΓÇö Macro utilities: `TEST_FLAG`, `SET_FLAG`, `MAX`, `MIN`, pinning/clamping macros
- **world.h** ΓÇö World coordinate types (`world_point2d`, `world_point3d`, `world_distance`, `angle`, `_fixed`), trigonometry, distance functions
- **dynamic_limits.h** ΓÇö Dynamic limit accessor `get_dynamic_limit()` for entity caps
- **shape_descriptors.h** ΓÇö Shape descriptor type and macros (`BUILD_DESCRIPTOR`, `GET_DESCRIPTOR_SHAPE`, etc.)
- **`<vector>`** ΓÇö STL vector for dynamic arrays (ObjectList, PolygonList, etc.)
- External symbols (defined elsewhere): Monster/projectile managers, rendering system, audio system, physics engine

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

# Source_Files/GameWorld/marathon2.cpp
## File Purpose

Central game loop and world state manager for the Marathon/Aleph One engine. Handles main tick simulation, action queue coordination, player movement prediction, level transitions, and trigger logic.

## Core Responsibilities

- **Main game loop** (`update_world`) ΓÇö orchestrates all per-tick entity and physics updates
- **Action queue management** ΓÇö merges input from real players, Lua scripts, and prediction systems
- **Client-side prediction** ΓÇö speculatively advances game state ahead of network confirmation, then rolls back when real updates arrive
- **Level transitions** ΓÇö `entering_map` and `leaving_map` coordinate resource loading/unloading and initialization
- **Polygon-based triggers** ΓÇö detects when entities cross boundaries and activates lights, platforms, monsters, items
- **Game state validation** ΓÇö calculates level completion conditions (extermination, exploration, retrieval, repair, rescue)
- **Damage calculations** ΓÇö applies difficulty modifiers and randomness

## External Dependencies

- **Geometry/Physics:** `map.h`, `world.h`, `flood_map.h`
- **Entities:** `monsters.h`, `projectiles.h`, `player.h`, `items.h`, `weapons.h`, `scenery.h`, `effects.h`
- **Systems:** `render.h`, `interface.h`, `network.h`, `network_games.h`, `SoundManager.h`, `Music.h`
- **Scripting:** `lua_script.h`, `lua_hud_script.h`
- **Input/Control:** `ActionQueues.h` (action queue system), input/output Lua and real player actions
- **Config/Profile:** `FilmProfile.h` (behavior flags for compatibility across game versions)
- **Misc:** `AnimatedTextures.h`, `ChaseCam.h`, `OGL_Setup.h`, `lightsource.h`, `media.h`, `platforms.h`, `fades.h`, `tags.h`, `motion_sensor.h`, `ephemera.h`, `interpolated_world.h`, `Plugins.h`, `Statistics.h`, `Console.h`, `Movie.h`, `game_window.h`, `screen.h`, `shell.h`, `SoundsPatch.h`

**Defined elsewhere:**
- `dynamic_world`, `static_world` ΓÇö global game/map state
- `GetGameQueue()`, `GetRealActionQueues()`, `GetLuaActionQueues()` ΓÇö action queue accessors
- `NetProcessMessagesInGame()`, `NetCheckWorldUpdate()`, `NetSync()`, `NetUpdateUnconfirmedActionFlags()`, `NetGetUnconfirmedActionFlag()`, `NetGetNetTime()` ΓÇö networking hooks
- `L_Call_Idle()`, `L_Call_PostIdle()`, `L_Call_Init()`, `L_Call_Cleanup()`, `L_Calculate_Completion_State()` ΓÇö Lua callbacks
- `update_lights()`, `update_medias()`, `update_platforms()`, `update_players()`, `move_projectiles()`, `move_monsters()`, `update_effects()` ΓÇö subsystem tick handlers
- `get_heartbeat_count()` ΓÇö frame-synchronization timer

# Source_Files/GameWorld/media.cpp
## File Purpose
Implements management of in-world liquids/media (water, lava, goo, sewage, jjaro). Handles instantiation, per-frame updates, property queries, serialization, and XML-based configuration of liquid behavior and effects.

## Core Responsibilities
- Create and maintain media instances (liquids) in the game world
- Calculate and update media heights based on dynamic light intensity
- Retrieve media-specific properties (damage, sounds, effects, textures, fade effects)
- Verify media availability in specific environment types
- Serialize/deserialize media state for save/load
- Parse and apply XML-based liquid configuration (MML)
- Evaluate whether media is dangerous to entities

## External Dependencies
- **map.h**: SLOT_IS_USED macro, damage_definition struct, dynamic_world, shape_descriptor, world types
- **effects.h**: Effect type enums (NUMBER_OF_EFFECT_TYPES)
- **fades.h**: Fade effect enums (NUMBER_OF_FADE_EFFECT_TYPES)
- **lightsource.h**: get_light_intensity() function
- **SoundManager.h**: (included but not directly used in this file)
- **InfoTree.h**: XML parsing class for MML support
- **Packing.h**: StreamToValue, ValueToStream macros for serialization
- **media_definitions.h** (not bundled): media_definition struct and media_definitions array
- **cseries.h**: Platform types, macros (FIXED_INTEGERAL_PART, WORLD_FRACTIONAL_PART, TRIG_SHIFT)
- **cmath functions**: trig tables (cosine_table, sine_table) assumed global

# Source_Files/GameWorld/media.h
## File Purpose
Header defining the media (liquid) system for the game engine. Media represents in-world liquids (water, lava, goo, etc.) with configurable properties including height, current, damage, and audio. Supports flexible media heights via light intensity mapping and extensible per-map media definitions via MML (Marathon Markup Language).

## Core Responsibilities
- Define media type enums (`_media_water`, `_media_lava`, `_media_goo`, `_media_sewage`, `_media_jjaro`)
- Define media flags (sound obstruction) and associated macros
- Declare `media_data` structure encoding all persistent media properties (32 bytes)
- Manage global `MediaList` vector of all active media instances
- Declare accessors and query functions for media properties
- Provide MML parsing and runtime update routines

## External Dependencies
- **Includes**: `<vector>` (C++ STL), `map.h` (provides `SLOT_IS_USED` macros and world types)
- **Types from elsewhere**: `damage_definition` (map.h), `shape_descriptor`, `world_distance`, `world_point2d`, `world_point3d`, `angle`, `_fixed` (world.h / type system)
- **Classes declared**: `InfoTree` (XML tree; defined elsewhere, used for MML parsing)
- **Macros used**: `TEST_FLAG16`, `SET_FLAG16`, `UNDER_MEDIA` (height test)

# Source_Files/GameWorld/media_definitions.h
## File Purpose
Defines static configuration data for all media types (water, lava, goo, sewage, Jjaro) in the game world. Each media type specifies visual effects, audio effects, damage properties, and rendering effects when submerged.

## Core Responsibilities
- Define `media_definition` struct for media type configuration
- Initialize static array `media_definitions[NUMBER_OF_MEDIA_TYPES]` with data for all 5 media types
- Specify detonation/splash visual effects for each media size (small, medium, large, emergence)
- Map media sounds (entering, exiting, walking, ambient)
- Define damage properties and frequency for harmful media
- Specify submerged fade/tint effects (color overlay when player underwater)

## External Dependencies
- **effects.h**: Effect type enumerations (`_effect_small_water_splash`, `_effect_under_water`, etc.)
- **fades.h**: Fade/tint effect types (`_effect_under_water`, `_effect_under_lava`, etc.)
- **media.h**: Media type enums (`_media_water`, `_media_lava`, etc.); `damage_definition` struct
- **SoundManagerEnums.h**: Sound type enumerations (`_snd_enter_water`, `_snd_walking_in_water`, etc.)

# Source_Files/GameWorld/monster_definitions.h
## File Purpose
Defines the data structures and enumerated constants that describe all monster types in the game engine. Contains class/faction relationships, behavioral flags, and a complete array of 40+ monster type definitions initialized with gameplay parameters (vitality, sounds, attacks, visual properties, etc.). The file serves as the authoritative data blueprint for monster instantiation and behavior.

## Core Responsibilities
- Define monster class enumerations and faction relationships (friends/enemies bitfields)
- Define monster behavioral flags (omniscience, invisibility, berserk, kamikaze, etc.)
- Define secondary constants: intelligence levels, door retry masks, and movement speeds
- Define `attack_definition` and `monster_definition` structures
- Provide complete initialization data for `original_monster_definitions` array (immutable master copy)
- Declare pack/unpack functions for serialization of monster definitions

## External Dependencies
- **Notable includes / imports:**
  - `effects.h` ΓÇö effect type constants (`_effect_fighter_blood_splash`, etc.) for impact effects
  - `items.h` ΓÇö item type constants for monster drops (`_i_magnum_magazine`, `_i_plasma_magazine`, etc.)
  - `map.h` ΓÇö type definitions: `world_distance`, `world_point3d`, `damage_definition`, shape descriptors
  - `monsters.h` ΓÇö external interface (not included but referenced in comments and prototypes)
  - `projectiles.h` ΓÇö projectile type constants used in `attack_definition` 
  - `SoundManagerEnums.h` ΓÇö sound ID constants (`_snd_fighter_activate`, `_snd_tick_chatter`, etc.)

- **Symbols defined elsewhere:**
  - `NUMBER_OF_MONSTER_TYPES` ΓÇö defined in `monsters.h`
  - `BUILD_COLLECTION()`, `FLAG()`, `UNONE`, `NONE` ΓÇö world/shape macros (likely from `map.h` or `world.h`)
  - World unit constants: `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH`, `WORLD_ONE_HALF`, `FIXED_ONE`, etc.

# Source_Files/GameWorld/monsters.cpp
## File Purpose
Implements core monster AI, physics, and lifecycle management for the Aleph One/Marathon game engine. Handles monster activation, pathfinding, combat, damage, death, and state transitions. Processes approximately 30 monsters per frame across target selection, movement, and attack execution.

## Core Responsibilities
- Manage monster state machine (modes: locked/lost/unlocked; actions: moving/attacking/dying)
- Update monster physics each frame (velocity, gravity, collision with terrain)
- Implement AI decision-making (target selection, line-of-sight, evasion)
- Pathfinding via flood-fill algorithm with cost functions
- Combat system (melee and ranged attack validation, projectile spawning)
- Monster activation/deactivation based on proximity and trigger conditions
- Difficulty scaling (drop rates, attack repetitions, monster promotion/demotion)
- Serialization for save/load functionality
- Lua scripting integration and event callbacks

## External Dependencies
- **map.h**: Polygon, line, object, endpoint data structures; flood_map; media queries
- **render.h**: view_data
- **interface.h**: Shape/animation data; collection loading
- **effects.h**: new_effect(); teleport effects
- **projectiles.h**: new_projectile()
- **player.h**: Player/monster interaction
- **platforms.h**: Platform accessibility checks
- **flood_map.h**: flood_map(), pathfinding cost functions
- **SoundManager.h**: LoadSound(), play_object_sound()
- **FilmProfile.h**: Compatibility flags (ketchup_fix, validate_random_ranged_attack, etc.)
- **lua_script.h**: L_Invalidate_Monster(), Lua monster_killed trigger
- **Packing.h**: Stream I/O helpers (StreamToValue, ValueToStream)
- **InfoTree.h**: XML parsing for MML
- **Logging.h**: logWarning()
- **monster_definitions.h**: Global monster_definitions array (imported, not defined here)

# Source_Files/GameWorld/monsters.h
## File Purpose
Defines the monster (NPC) system for the Marathon game engine, including type definitions, state structures, behavior modes, and function prototypes for managing monster lifecycle, AI, activation, and combat interactions.

## Core Responsibilities
- Define monster type enumeration (40+ types: marines, cyborgs, hunters, yetis, etc.)
- Define monster state structure (`monster_data`, 64 bytes) for per-instance data
- Manage monster behavior modes (locked, losing lock, running, etc.) and actions (stationary, moving, attacking, dying)
- Provide bitfield macros for monster flags (active, berserk, idle, blind, deaf, etc.)
- Declare lifecycle functions (spawn, remove, activate, deactivate)
- Declare movement and AI functions (pathfinding, target selection, activation)
- Declare damage and collision functions (radius damage, melee, movement validation)
- Support dynamic limits on monster counts and collision buffers

## External Dependencies
- **`dynamic_limits.h`:** Runtime configuration of monster counts and buffer sizes.
- **`world.h`:** Spatial types (`world_point3d`, `world_distance`, `angle`, `world_location3d`), distance/angle math.
- **`<vector>`:** STL container for `MonsterList`.
- **`InfoTree`:** (forward declared) XML/config tree for MML parsing (`parse_mml_monsters()`, `parse_mml_damage_kicks()`).
- **`struct damage_definition`, `struct object_location`:** Defined elsewhere; used in function signatures.

# Source_Files/GameWorld/pathfinding.cpp
## File Purpose
Implements waypoint pathfinding for monsters and NPCs. Creates sequences of 2D waypoints that guide entities across polygon-based map geometry using breadth-first flood-fill algorithms. Handles both destination-based paths and random biased paths for unreachable targets.

## Core Responsibilities
- Allocate and manage a global pool of reusable path structures
- Generate paths via flood-fill from source to destination polygon
- Calculate waypoint positions at shared polygon boundaries with minimum separation constraints
- Support random path generation when destinations are unreachable
- Track stepwise traversal along an active path
- Delete and reset paths for reallocation across game ticks
- Validate path consistency in debug builds

## External Dependencies
- **map.h** ΓÇô Polygon/line/endpoint accessors; `find_shared_line()`, `get_line_data()`, `get_endpoint_data()`
- **flood_map.h** ΓÇô `flood_map()`, `reverse_flood_map()`, `flood_depth()`, `choose_random_flood_node()`; `cost_proc_ptr` typedef
- **dynamic_limits.h** ΓÇô `get_dynamic_limit()`
- **cseries.h** ΓÇô Utilities (`obj_set`, `objlist_copy`), assertions
- Standard C/C++ ΓÇô `<string.h>` (memcmp), `<limits.h>` (INT32_MAX), `<stdlib.h>`

# Source_Files/GameWorld/physics.cpp
## File Purpose

Core physics engine implementing all player movement, gravity, collision detection, and aim input processing for a Marathon-engine game. Handles frame-by-frame position updates, velocity management, and interaction with the game world geometry.

## Core Responsibilities

- Initialize and update player physics state each tick (30/second)
- Compute and apply velocity, acceleration, gravity, and external forces
- Process player input (movement, aiming, action flags) into physics changes
- Detect and resolve collisions with walls, objects, and polygon boundaries
- Support multiple physics models (editor, earth gravity, low gravity)
- Manage high-precision virtual aim tracking for network clients with frame interpolation
- Handle polygon height changes affecting player elevation
- Serialize physics constants to/from byte streams for saves and networking

## External Dependencies

- **cseries.h** ΓÇô Basic types, macros, fixed-point utilities
- **render.h** ΓÇô Camera/view data (FOV, screen dimensions)
- **map.h** ΓÇô World geometry: polygons, endpoints, lines, sides; collision functions; media definitions
- **player.h** ΓÇô `player_data`, `physics_variables`, `physics_constants` struct definitions; action flag enums
- **interface.h** ΓÇô Game state, preferences access, player count
- **monsters.h** ΓÇô Monster data, definitions; `bump_monster()`, `legal_player_move()`
- **preferences.h** ΓÇô Input preferences: mouse sensitivity, classic aim behavior, auto-recenter flag
- **monster_definitions.h** ΓÇô Monster type flags (e.g., `_monster_can_grenade_climb`)
- **media.h** ΓÇô Media (liquid) height queries
- **ChaseCam.h** ΓÇô Third-person camera support (DROP_DEAD_HEIGHT adjustment)
- **Packing.h** ΓÇô Serialization helpers (`StreamToValue`, `ValueToStream`)

**Trigonometric lookups:** `cosine_table[]`, `sine_table[]` (pre-computed for fixed-angle indices)

# Source_Files/GameWorld/physics_models.h
## File Purpose
Defines the physics models and constants for character movement (walking and running) in the game engine. Provides data structures to parameterize velocity, acceleration, angular movement, and collision geometry for player characters. Declares serialization functions to pack/unpack physics constants from binary data.

## Core Responsibilities
- Define enumeration of physics model types (`_model_game_walking`, `_model_game_running`)
- Declare `physics_constants` struct containing all physics parameters (velocities, accelerations, angular properties, dimensions)
- Provide hardcoded default physics constants for two movement models
- Declare serialization/deserialization functions for physics data (binary I/O)
- Declare initialization function for physics constants at engine startup

## External Dependencies
- **Includes:** `world.h` ΓÇö provides fixed-point arithmetic (`_fixed` type), angle constants (`QUARTER_CIRCLE`), distance types (`world_distance`)
- **External symbols:** `FIXED_ONE`, `QUARTER_CIRCLE` macros from world.h; `_fixed`, `uint8` types from world.h/cstypes.h
- **Implicit:** Standard C integer types (`size_t`, `uint8`)

# Source_Files/GameWorld/placement.cpp
## File Purpose
Manages placement and respawning of monsters and items across game levels. Loads placement data from maps, places initial objects at level start, and periodically recreates monsters/items based on difficulty and replenishment settings.

## Core Responsibilities
- Load and unpack placement configuration data from WAD files
- Place initial monsters and items on level load
- Periodically respawn/recreate monsters and items according to min/max/random counts
- Track active object counts per type
- Find valid random spawn locations (respecting collision, visibility, polygon type)
- Select random player starting locations weighted by distance to monsters/players
- Validate polygon eligibility for object placement

## External Dependencies
- **map.h:** Polygon/map geometry (polygon_data, object_location, world_point2d/3d), map queries (find_new_object_polygon, point_in_polygon, find_center_of_polygon), object creation (new_map_object), visibility checks (point_is_player_visible, point_is_monster_visible, get_polygon_data, get_object_data).
- **monsters.h:** Monster creation/activation (new_monster, activate_monster, find_closest_appropriate_target), loading (mark_monster_collections, load_monster_sounds).
- **items.h:** Item creation (new_item).
- **FilmProfile.h:** Compatibility flag `film_profile.initial_monster_fix` (controls whether initial monsters spawn based on film mode).
- **cseries.h:** Utility (cseries/csmacros.h for obj_clear, objlist_clear, TEST_FLAG, etc.).

# Source_Files/GameWorld/platform_definitions.h
## File Purpose
Defines default sound and behavioral configurations for all platform types in the game world. Provides a lookup table that maps each platform type (doors, platforms, etc.) to its associated sound effects, damage properties, and initial behavior flags. This file is part of the Aleph One engine (Marathon mod) and serves as initialization data for platform behavior.

## Core Responsibilities
- Enumerate platform sound codes (starting, stopping, obstructed, uncontrollable)
- Define the `platform_definition` struct to hold per-platform-type configuration
- Populate the `platform_definitions` array with concrete data for all 9 platform types
- Associate each platform type with its audio assets (opening, closing, ambient, obstruction sounds)
- Set default behavior flags for each platform type (speed, delay, control modes, damage)
- Provide damage characteristics (damage type and thresholds) for each platform type

## External Dependencies

### Includes
- `platforms.h` ΓÇö Provides enums for platform types (`_platform_is_spht_door`, `_platform_is_pfhor_door`, etc.), speed/delay constants, flag macros, and struct definitions (`static_platform_data`, `damage_definition`)
- `SoundManagerEnums.h` ΓÇö Provides sound ID enumerations (e.g., `_snd_spht_door_opening`, `_ambient_snd_spht_door`)

### Defined Elsewhere
- `static_platform_data` ΓÇö struct for platform defaults (speed, delay, flags, polygon index)
- `damage_definition` ΓÇö struct for damage properties (type, minimum damage, maximum damage, threshold)
- Sound enums (`_snd_*`, `_ambient_snd_*`) ΓÇö from SoundManagerEnums.h
- Platform type enums (`_platform_is_*`) and flag macros (`FLAG()`) ΓÇö from platforms.h

# Source_Files/GameWorld/platforms.cpp
## File Purpose
Implements dynamic platform and door mechanics for the game world. Manages platform creation, movement, collision detection, state changes, adjacent platform cascades, and interaction with world geometry including media. Supports both player and monster interaction with platforms and provides serialization/XML configuration.

## Core Responsibilities
- Create and initialize platforms in the world (`new_platform`)
- Update platform movement and state each frame (`update_platforms`)
- Activate/deactivate platforms and cascade state to adjacent platforms (`set_platform_state`, `set_adjacent_platform_states`)
- Calculate and maintain platform height extrema for movement bounds (`calculate_platform_extrema`)
- Detect and handle collisions/obstructions during platform movement
- Update geometry (endpoint heights, line solidity/transparency, texture coordinates) when platform heights change
- Manage platform-media interactions (water/lava submersion tracking)
- Play contextual sounds (starting, stopping, blocked, uncontrollable)
- Evaluate AI accessibility to/from platforms (`monster_can_enter_platform`, `monster_can_leave_platform`)
- Handle player activation and control of platforms (`player_touch_platform_state`)
- Serialize/deserialize platform data for save/load
- Parse and apply XML configuration for platform type definitions (`parse_mml_platforms`)

## External Dependencies
- **world.h**: World coordinate types (`world_distance`, `world_point2d`, `world_point3d`), geometry utilities
- **map.h**: Map structures (`polygon_data`, `line_data`, `endpoint_data`, `side_data`), polygon/line/endpoint accessors, `dynamic_world`, `change_polygon_height`
- **platforms.h**: Type definitions (`platform_data`, `static_platform_data`), flag macros, function prototypes
- **lightsource.h**: `set_light_status` for platform-controlled lights
- **SoundManager.h**: Sound playback (`play_polygon_sound`, `SoundManager::instance()->CauseAmbientSoundSourceUpdate`)
- **player.h**: `try_and_subtract_player_item` for key consumption, player item access
- **media.h**: `get_media_data` for media height queries
- **lua_script.h**: `L_Call_Platform_Activated` (Pfhortran scripting hook)
- **InfoTree.h**: XML parsing for MML configuration
- **items.h**, **Packing.h**: Damage definition parsing (included but minimal use visible)
- **editor.h**: `MARATHON_ONE_DATA_VERSION` constant for version-specific unpacking

# Source_Files/GameWorld/platforms.h
## File Purpose

Defines the platform system for the Marathon/Aleph One game engine. Platforms are dynamic geometry elements (doors, rising platforms, etc.) that can move, activate based on events, and respond to player/monster interaction.

## Core Responsibilities

- Define 9 platform type constants and speed/delay enumerations
- Manage 32 static (design-time) and 16 dynamic (runtime) state flags via bitwise macros
- Define `static_platform_data` (32-byte design template) and `platform_data` (140-byte runtime instance) structures
- Declare platform lifecycle functions: creation, activation, state changes, media interaction
- Declare platform accessibility queries for pathfinding (can monsters traverse?)
- Declare serialization and MML-based data loading
- Maintain the global `PlatformList` vector as the authority on all map platforms

## External Dependencies

- **map.h**: polygon_data, line_data, endpoint_data, world_distance, world_point3d, side_data, MAXIMUM_VERTICES_PER_POLYGON
- **Macros (via map.h or csmacros.h):** TEST_FLAG16, SET_FLAG16, TEST_FLAG32, SET_FLAG32
- **Constants:** WORLD_ONE, TICKS_PER_SECOND
- **MML parsing:** InfoTree (likely from XML/data module)
- **Bungie/Aleph One licensing:** GNU GPL v3

# Source_Files/GameWorld/player.cpp
## File Purpose
Core player management system for the Marathon game engine. Handles all aspects of player lifecycle including creation, physics, damage, death/revival, equipment management, and network serialization. Supports configurable player attributes via MML (Marathon Markup Language).

## Core Responsibilities
- Player creation, initialization, and destruction across single-player and multiplayer games
- Per-frame player state updates including physics, oxygen/energy depletion, and animation
- Damage application, death mechanics, and player respawning with optional penalties
- Powerup acquisition, duration tracking, and visual effect management
- Player equipment (weapons, items) and initial inventory distribution
- Teleportation (level-based and intra-level) handling and animation
- Action queue management and hotkey sequence decoding for weapon selection
- Terminal interaction mode with time-stop in solo play
- Player data serialization/deserialization for save games and network transmission
- MML-driven configuration of player settings, powerups, initial items, and damage responses

## External Dependencies
- **map.h**: polygon_data, object_data, damage_definition, map object functions (new_map_object, translate_map_object, etc.), world geometry queries
- **monsters.h / monster_definitions.h**: monster_data, MONSTER_IS_PLAYER, monster_definition, damage types, monster flags
- **weapons.h**: weapon management, update_player_weapons, initialize_player_weapons, discharge_charged_weapons, player weapon mode queries
- **items.h**: item type constants, item functions (swipe_nearby_items, give/drop items)
- **projectiles.h**: detonate_projectile (for netdead effect)
- **SoundManager.h**: play_object_sound, sound effect enums
- **interface.h**: collection management, screen_printf, update_interface, mark_display_dirty functions
- **network.h / network_games.h**: multiplayer game state, player_killed_player callback
- **lua_script.h**: L_Call_Player_Damaged, L_Call_Player_Killed, L_Call_Player_Revived (Lua event callbacks)
- **ChaseCam.h**: ChaseCam_Reset for third-person view handling
- **ActionQueues.h**: dequeueActionFlags, modifyActionFlags, action queue management
- **InfoTree.h**: XML configuration parsing infrastructure

# Source_Files/GameWorld/player.h
## File Purpose
Header file defining player entity data structures, constants, and function prototypes for the Aleph One game engine. Manages player state (position, health, weapons, powerups), physics variables, and action input handling. Supports single-player and networked multiplayer modes.

## Core Responsibilities
- Define player state structure (`player_data`) with position, health, weapons, and metadata
- Define physics simulation variables for movement, orientation, and collision
- Declare action flag enums and bitfield manipulation macros for input encoding
- Provide player configuration via `player_settings_definition` (settable via MML)
- Declare initialization, update, and damage functions for per-frame game loop
- Manage global player arrays and accessor functions
- Handle player-specific powerup logic and weapon/item management

## External Dependencies
- **Notable includes:** `cseries.h` (core types, macros), `world.h` (coordinate types, angle, world_point3d), `map.h` (world geometry, WORLD_ONE constant), `weapons.h` (weapon type enums)
- **Defined elsewhere:**
  - `ActionQueues`, `ModifiableActionQueues` (forward declarations; defined elsewhere)
  - `InfoTree` (forward declaration; XML parsing support)
  - All `unpack_*` / `pack_*` functions (implement serialization, declared here)
  - Weapon and physics constants from included headers

# Source_Files/GameWorld/projectile_definitions.h
## File Purpose

Defines the static data structures and configurations for all projectile types in the game engine. Contains the master `projectile_definition` struct and initializes a global array with concrete parameters (damage, speed, effects, sounds, flags) for every projectile type in the game.

## Core Responsibilities

- Define projectile behavior flags (guided, gravity-affected, rebounds, penetration, etc.) as bit-flags
- Declare the `projectile_definition` struct that aggregates all projectile properties
- Initialize `original_projectile_definitions[]` with complete configuration for ~40 projectile types
- Provide serialization/deserialization entry points for projectile definition data
- Support mod/customization by maintaining a mutable copy of projectile definitions

## External Dependencies

- **effects.h** ΓÇô Effect type constants (`_effect_rocket_explosion`, `_effect_grenade_explosion`, etc.)
- **map.h** ΓÇô `damage_definition` struct; world coordinate types; `WORLD_ONE` scale constant
- **media.h** ΓÇô Media detonation effect types
- **projectiles.h** ΓÇô Projectile type enum (`NUMBER_OF_PROJECTILE_TYPES`); `projectile_data` struct
- **SoundManagerEnums.h** ΓÇô Sound event constants (`_snd_rocket_flyby`, `_snd_fusion_flyby`, etc.); frequency constants (`_normal_frequency`, `_higher_frequency`)

---

**Notes on Projectile Flags:**
The file defines ~24 flag bits controlling projectile behavior: guidance, gravity interaction, media penetration, collision handling, detonation modes, and visual effects. Most flags are mutually exclusive or apply to specific projectile families (alien vs. player weapons, melee vs. ranged). Two specialized flags (`_penetrates_media_boundary`, `_passes_through_objects`) enable SMG bullets to enter/exit liquids without detonatingΓÇöa feature added in Feb 2000.

# Source_Files/GameWorld/projectiles.cpp
## File Purpose
Implements projectile simulation for the Marathon/Aleph One game engine. Handles creation, movement, collision detection, damage application, and lifecycle management of all projectile objects in the game world.

## Core Responsibilities
- Create and initialize new projectiles when weapons fire
- Simulate projectile movement each frame, applying physics (gravity, guided targeting)
- Detect collisions with landscape, media, monsters, and scenery
- Calculate and apply damage to hit targets
- Create detonation effects and handle penetration of media boundaries
- Remove projectiles when they expire or hit obstacles
- Manage guided projectile targeting and trajectory adjustments
- Serialize/deserialize projectile state for saving and networking
- Pre-flight validation of projectile placement before creation

## External Dependencies
- **map.h**: polygon_data, line_data, endpoint_data, get_polygon_data, get_line_data, find_line_crossed_leaving_polygon, find_line_intersection, find_adjacent_polygon, new_map_object3d, translate_map_object, remove_map_object
- **effects.h**: new_effect, get_effect_data, mark_effect_collections
- **monsters.h**: damage_monsters_in_radius, damage_monster, get_monster_data, get_monster_dimensions, get_monster_impact_effect, get_monster_melee_impact_effect, possible_intersecting_monsters
- **player.h**: try_and_add_player_item, monster_index_to_player_index
- **scenery.h**: get_scenery_dimensions, damage_scenery
- **media.h**: get_media_data, get_media_detonation_effect
- **items.h**: get_item_shape, new_item
- **SoundManager.h**: SoundManager::instance()->LoadSound
- **lua_script.h**: L_Call_Projectile_Created, L_Call_Projectile_Detonated, L_Invalidate_Projectile
- **projectile_definitions.h** (included): projectile_definitions array, original_projectile_definitions

# Source_Files/GameWorld/projectiles.h
## File Purpose
Defines the projectile system for the Aleph One game engine, managing all player-fired and enemy-fired projectiles. Provides data structures, type definitions, and APIs for creating, simulating, and destroying projectiles throughout the game world.

## Core Responsibilities
- Define 40+ projectile types (rockets, grenades, bullets, bolts, hummers, etc.)
- Manage the `projectile_data` structure (32-byte state record per projectile)
- Maintain a dynamic list of active projectiles in the game world
- Provide flag macros for tracking projectile state (flyby sound, damage infliction, media crossing)
- Expose functions for projectile lifecycle: creation, translation, collision detection, detonation, and removal
- Handle projectile-specific features: guided targeting, damage scaling, contrail effects
- Support serialization via packing/unpacking functions

## External Dependencies
- **dynamic_limits.h**: `get_dynamic_limit(_dynamic_limit_projectiles)` to retrieve max projectile count
- **world.h**: angle, world_point3d, world_distance, _fixed types; coordinate math and rotation
- **&lt;vector&gt;**: STL vector for dynamic projectile list
- **Presumed elsewhere**: Object system (object_index), polygon/polygon system (polygon_index), collision/physics, effects/sounds, monster types

# Source_Files/GameWorld/scenery.cpp
## File Purpose
Manages scenery objectsΓÇöstatic or animated decorative/environmental world elements. Handles creation, animation updates, destruction effects, and configuration via Marathon Map Language (MML) definitions.

## Core Responsibilities
- Create scenery objects with proper flags and physics properties
- Maintain and update animations for dynamic scenery each game tick
- Apply damage to scenery, including destruction transitions and effects
- Load and apply MML-based scenery definition overrides
- Query scenery dimensions, collections, and animation properties

## External Dependencies
- **map.h:** `new_map_object()`, `get_object_data()`, `animate_object()`, `randomize_object_sequence()`, `MAXIMUM_OBJECTS_PER_MAP`, object owner/flag macros.
- **effects.h:** `new_effect()`.
- **scenery.h:** Header declaring public interface (not shown).
- **InfoTree.h:** XML configuration parsing.
- Indirect: **dynamic_limits.h** (via get_dynamic_limit, not directly visible).
- **scenery_definitions.h:** External array `scenery_definitions` and constant `NUMBER_OF_SCENERY_DEFINITIONS`.

# Source_Files/GameWorld/scenery.h
## File Purpose
Header declaring functions for managing scenery (static/decorative objects) in the game world. Supports scenery creation, animation, damage, shape randomization, and data-driven configuration via MML. Part of the Aleph One/Marathon engine.

## Core Responsibilities
- Initialize and manage scenery object lifecycle
- Animate scenery and randomize visual variants
- Track and apply damage to scenery objects
- Retrieve scenery collection data (sprite/shape resources)
- Parse and reset MML-based scenery configuration
- Support Lua scripting for runtime scenery manipulation

## External Dependencies
- `#include "world.h"` ΓÇö Provides `world_distance`, `world_point3d`, vector/point types
- `object_location` ΓÇö Struct defining 3D position and polygon; defined elsewhere
- `InfoTree` ΓÇö XML/MML document tree class; likely from parser module
- Primitive types: `short`, `bool`

# Source_Files/GameWorld/scenery_definitions.h
## File Purpose
Defines static scenery object types (environmental decorations and obstacles) for the game world. Contains a 61-entry array of pre-configured scenery definitions organized by environment theme (lava, water, sewage, alien, Jjaro), each specifying visual representation, physical properties, and destruction behavior.

## Core Responsibilities
- Define the `scenery_definition` struct describing individual scenery object properties
- Provide scenery behavior flags (solid, animated, destroyable)
- Maintain a static array of 61 pre-defined scenery instances for world placement
- Map scenery objects to graphical shape descriptors and collections
- Configure destruction effects and post-destruction replacement shapes

## External Dependencies
- `effects.h`: Effect type enums (`_effect_lava_lamp_breaking`, `_effect_water_lamp_breaking`, etc.)
- `shape_descriptors.h`: `shape_descriptor` typedef; `BUILD_DESCRIPTOR(collection, shape)` macro
- `world.h`: `world_distance` typedef; distance constants (`WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH`)


# Source_Files/GameWorld/TickBasedCircularQueue.h
## File Purpose
Header-only template library implementing tick-based circular queues for the Aleph One game engine. Provides reader-writer synchronized circular buffers keyed by game tick, supporting concurrent access by separate reader and writer threads, with options for queue duplication and mutable element access.

## Core Responsibilities
- Define abstract write-only interface (`WritableTickBasedCircularQueue`) for enforcing producer-consumer contracts
- Implement concrete circular queue with tick-indexed storage and wrap-around buffer management
- Support broadcast writes via queue duplication (`DuplicatingTickBasedCircularQueue`)
- Enable post-enqueue element mutation under careful synchronization
- Manage capacity constraints to prevent buffer overruns in concurrent context
- Handle tick wrapping via modulo arithmetic for infinite game tick sequences

## External Dependencies
- `cseries.h`: Type definitions (`int32`, `int16`), assertion macros
- `<set>`: STL `std::set` for child queue collection in `DuplicatingTickBasedCircularQueue`
- Standard C++ assert; modulo operator for buffer wrapping

# Source_Files/GameWorld/weapon_definitions.h
## File Purpose
Defines all weapon properties, configurations, and constants for the game engine. Provides data structures for weapon mechanics including triggers, animations, shell casings, and sound effects. Contains both the definition schema and hardcoded definitions for all player weapons.

## Core Responsibilities
- Define weapon classification system (melee, normal, dual-function, two-fisted, multipurpose)
- Enumerate weapon behavioral flags (automatic, overload, disappears after use, etc.)
- Define animation state indices for weapon in-hand rendering and shell casings
- Specify trigger mechanics (rounds per magazine, ammo type, firing rate, charging, recoil)
- Specify weapon visual & audio properties (firing light intensity, shape indices, sounds)
- Provide complete weapon definition array with per-weapon configuration for all game weapons
- Support serialization/deserialization of weapon data for save/load

## External Dependencies

**Notable includes:**
- `items.h` ΓÇö for item type constants (`_i_knife`, `_i_magnum_magazine`, etc.)
- `map.h` ΓÇö for `world_distance` type and `TICKS_PER_SECOND` constant
- `projectiles.h` ΓÇö for projectile type constants (`_projectile_pistol_bullet`, `_projectile_rocket`, etc.)
- `SoundManagerEnums.h` ΓÇö for sound constants (`_snd_magnum_firing`, `_snd_assault_rifle_shell_casings`, etc.)
- `weapons.h` ΓÇö for weapon enumeration constants and related structures

**External symbols used but not defined:**
- Type: `_fixed` (fixed-point arithmetic type, likely from map.h or global headers)
- Type: `world_distance` (distance measurement type)
- Constants: `FIXED_ONE`, `FIXED_ONE_HALF`, `FIXED_ONE/n` (fixed-point arithmetic macros)
- Constants: Sound indices (e.g., `_snd_magnum_firing`) and projectile types (e.g., `_projectile_pistol_bullet`)
- Constants: Item type indices (e.g., `_i_magnum`, `_i_assault_rifle_magazine`)

# Source_Files/GameWorld/weapons.cpp
## File Purpose
Implements the weapon system for the Aleph One game engine, managing player weapon state machines, firing, reloading, animation, and shell casing physics. Handles all aspects of weapon selection, switching, ammo management, and state transitions.

## Core Responsibilities
- Initialize and manage weapon system for all players
- Update weapon state machines each frame (charging, firing, reloading, animations)
- Handle weapon firing events and projectile creation
- Manage ammo counts and reload mechanics for primary/secondary triggers
- Implement shell casing physics and interpolation for rendering
- Support weapon selection and automatic weapon switching
- Handle two-fisted and dual-function weapon behaviors
- Serialize/deserialize weapon data for save games
- Parse and apply MML configuration for weapon constants

## External Dependencies
- **Includes**: `map.h`, `projectiles.h`, `player.h`, `SoundManager.h`, `interface.h`, `items.h`, `monsters.h`, `Packing.h`, `shell.h`, `weapon_definitions.h`, `InfoTree.h`
- **External symbols**: 
  - `weapon_definitions`, `shell_casing_definitions`, `weapon_ordering_array` (defined elsewhere, probably weapons.h)
  - `new_projectile()` (projectiles.c)
  - `SoundManager::instance()->PlaySound()` (sound system)
  - `get_player_data()`, `dynamic_world` (player/map system)
  - `mark_collection_for_loading/unloading()` (shape/resource system)
  - `get_item_kind()` (items.c)

# Source_Files/GameWorld/weapons.h
## File Purpose
Header for the weapon management system in Aleph One (Marathon engine). Defines structures, enumerations, and function prototypes for managing player weapons, ammunition, firing state, display rendering, and serialization.

## Core Responsibilities
- Define weapon types and pseudo-weapons (fist, pistol, plasma pistol, assault rifle, SMG, shotgun, flamethrower, alien shotgun, missile launcher, etc.)
- Track per-player weapon state: current weapon, desired weapon, ammo counts, trigger state, animation frame
- Manage weapon display information for rendering in game window (collection, shape index, positioning, transfer mode)
- Handle shell casing rendering and animation
- Initialize weapons for new games and new players
- Update weapons each frame based on player input (action flags)
- Serialize/deserialize weapon state and definitions (pack/unpack functions)
- Load/unload weapon resource collections
- Process ammo pickups and reloading
- Support XML/MML configuration of weapon parameters
- Query weapon state for UI and rendering systems

## External Dependencies
- **cstypes.h** (bundled): Provides `_fixed`, `uint8`, `uint16`, `uint32`, `int16`, `int32`
- **InfoTree** class: (defined elsewhere) Used for XML/MML parsing
- **Resource system:** Weapon collections managed by external code (see `mark_weapon_collections`)
- **Projectile system:** Receives feedback via `player_hit_target()` (defined elsewhere)
- **Comment reference:** lua_script.cpp accesses shell casing constants

# Source_Files/GameWorld/world.cpp
## File Purpose
Core world geometry and mathematics module for the Aleph One game engine (Marathon mod). Provides trigonometric lookup tables, point transformations (translation/rotation in 2D/3D), distance calculations, angle normalization, and deterministic random number generation. Supports both legacy (Marathon 2) and modern (Aleph One) physics implementations for accurate long-distance calculations and film playback.

## Core Responsibilities
- Build and manage global sine/cosine/tangent lookup tables for fixed-point trigonometry
- Normalize angles to valid range [0, NUMBER_OF_ANGLES) = [0, 512) = [0, 2╧Ç)
- Translate points in 2D/3D space by distance and angle(s)
- Rotate and transform points around origins in 2D/3D (relative/absolute coordinate systems)
- Calculate Euclidean distances between points with overflow clamping
- Compute arctangent angles from (x, y) coordinates via binary search (Aleph One) or linear search (Marathon 2)
- Generate deterministic pseudo-random numbers with separate global and local LFSR seeds
- Support long-distance physics via overflow-bit packing in flags uint16
- Implement integer square root (isqrt) for distance calculations without floating-point

## External Dependencies
- **Includes:** cseries.h (platform), world.h (constants, declarations), FilmProfile.h (film_profile global), stdlib.h (malloc), math.h (cos, sin, atan), limits.h (INT16_MAX, INT32_MIN).
- **Macros:** TRIG_SHIFT, TRIG_MAGNITUDE, NUMBER_OF_ANGLES, NORMALIZE_ANGLE, GUESS_HYPOTENUSE, QUARTER_CIRCLE, HALF_CIRCLE, THREE_QUARTER_CIRCLE, EIGHTH_CIRCLE.
- **External globals:** `film_profile` (struct FilmProfile).

# Source_Files/GameWorld/world.h
## File Purpose
Defines foundational world coordinate systems, geometric types, and mathematical utilities for the Aleph One game engine. Provides fixed-point and integer-based coordinate representations (world_distance vs. long int32), trigonometric helpers, and spatial transformation functions for 2D/3D geometry operations.

## Core Responsibilities
- Define world coordinate distance types and fractional-bit constants (WORLD_FRACTIONAL_BITS = 10)
- Provide dual coordinate hierarchies: `world_*` (int16-based, lower precision) and `long_*` (int32-based, higher precision) for 2D/3D points and vectors
- Declare trigonometric tables (sine/cosine) and angle normalization macros
- Declare spatial transformation functions (rotation, translation, composition) for both 2D and 3D points
- Declare distance calculation functions and integer square root
- Declare world location structures that bundle position, orientation, polygon index, and velocity
- Provide overflow handling for long-distance coordinates via flags-based kludges

## External Dependencies
- **Includes:** `cstypes.h` (for base integer types and `_fixed`), `<tuple>` (for `std::tie` in `world_location3d` equality operators)
- **Defined elsewhere:** `cosine_table`, `sine_table` (extern symbols from WORLD.C); all function implementations (`build_trig_tables`, `rotate_point*`, `translate_point*`, `transform_point*`, `arctangent`, `global_random`, `local_random`, `distance*`, `isqrt`, overflow functions) are in WORLD.C


