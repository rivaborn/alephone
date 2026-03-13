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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| object_data | struct | Game objects (scenery, items, effect carriers) |
| polygon_data | struct | Room/area geometry with floor, ceiling, properties |
| line_data | struct | Wall boundary segments with adjacency info |
| side_data | struct | Wall surface textures and control panels |
| endpoint_data | struct | Map vertices with height and solidity flags |
| monster_data | struct | Monster entity state (from monsters.h) |
| projectile_data | struct | Projectile entity state (from projectiles.h) |
| effect_data | struct | Visual/audio effect state (from effects.h) |
| static_data | struct | Immutable level properties (music, mission type) |
| dynamic_data | struct | Mutable game state (entity counts, scores) |
| ambient_sound_image_data | struct | Non-directional ambient sound sources |
| random_sound_image_data | struct | Periodic/directional random sound effects |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| static_world | static_data* | global | Level-static properties |
| dynamic_world | dynamic_data* | global | Game state counters and tick info |
| ObjectList, MonsterList, ProjectileList, EffectList | vector<> | global | Entity storage (vectors replacing fixed arrays) |
| EndpointList, LineList, SideList, PolygonList | vector<> | global | Map geometry storage |
| AmbientSoundImageList, RandomSoundImageList | vector<> | global | Sound source storage |
| Environments | static short[5][7] | static | Texture collections per environment theme |
| LandscapesLoaded | bool | global | M2 landscape loading flag for M1 compat |
| LoadedWallTexture | short | global | Index of primary wall texture for infravision |
| IntersectedObjects | static vector<short> | static | Growable buffer for collision queries |
| sDeferredObjectListInsertions | static list<> | static | Pending polygon list operations (prediction) |

## Key Functions / Methods

### allocate_map_memory
- Signature: `void allocate_map_memory(void)`
- Purpose: One-time allocation of dynamic memory for map structures at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes static_world and dynamic_world pointers; clears memory via obj_clear()
- Calls: new operator, obj_clear()

### initialize_map_for_new_level
- Signature: `void initialize_map_for_new_level(void)`
- Purpose: Reset all transient map state when loading a new level while preserving game progress
- Inputs: None
- Outputs/Return: None (modifies dynamic_world)
- Side effects: Preserves player_count, tick_count, random_seed, civilian stats; clears all entities, geometry, and sounds
- Calls: obj_clear(), objlist_clear(), Console::instance()->clear_saves()

### new_map_object3d
- Signature: `short new_map_object3d(world_point3d *location, short polygon_index, shape_descriptor shape, angle facing)`
- Purpose: Create a new object in the world at exact 3D coordinates
- Inputs: Location (3D point), polygon, shape descriptor, facing angle
- Outputs/Return: Object index (NONE if table full)
- Side effects: Initializes object_data; inserts at head of polygon's object linked-list
- Calls: _new_map_object(), get_polygon_data(), get_object_data()
- Notes: Sibling functions new_map_object2d() and new_map_object() provide convenience variants

### translate_map_object
- Signature: `bool translate_map_object(short object_index, world_point3d *new_location, short new_polygon_index)`
- Purpose: Move an object to a new location, automatically updating polygon ownership and parasitic objects
- Inputs: Object index, new location (3D), new polygon index (NONE = auto-calculate)
- Outputs/Return: true if polygon changed
- Side effects: Updates object->location and polygon; removes from old polygon list, adds to new; moves parasitic objects
- Calls: find_line_crossed_leaving_polygon(), find_adjacent_polygon(), remove_object_from_polygon_object_list(), add_object_to_polygon_object_list(), get_polygon_data()
- Notes: If new_polygon_index=NONE, traces ray to find destination polygon; if not found, centers object in old polygon

### remove_map_object
- Signature: `void remove_map_object(short object_index)`
- Purpose: Completely remove an object and clean up its state
- Inputs: Object index
- Outputs/Return: None
- Side effects: Removes from polygon's object list; deletes parasitic object if present; invalidates Lua references
- Calls: get_object_data(), get_polygon_data(), L_Invalidate_Object(), MARK_SLOT_AS_FREE()

### perform_deferred_polygon_object_list_manipulations
- Signature: `void perform_deferred_polygon_object_list_manipulations(void)`
- Purpose: Execute pending object list insertions (scheduled by deferred_add_object_to_polygon_object_list)
- Inputs: None (uses static sDeferredObjectListInsertions)
- Outputs/Return: None
- Side effects: Modifies polygon object lists; empties deferred list
- Calls: get_object_data(), get_polygon_data()
- Notes: Supports netplay/prediction by deferring insertions until all objects are in correct polygons

### keep_line_segment_out_of_walls
- Signature: `bool keep_line_segment_out_of_walls(short polygon_index, world_point3d *p0, world_point3d *p1, world_distance maximum_delta_height, world_distance height, world_distance *adjusted_floor_height, world_distance *adjusted_ceiling_height, short *supporting_polygon_index)`
- Purpose: Clip a movement line segment to prevent wall and obstacle penetration
- Inputs: Polygon, line endpoints (p0ΓåÆp1), height constraints, max step-up height
- Outputs/Return: true if clipping occurred; adjusted heights/supporting polygon via pointers
- Side effects: Modifies p1 coordinates (destructive)
- Calls: find_line_crossed_leaving_polygon(), find_adjacent_polygon(), get_polygon_data(), get_line_data(), get_endpoint_data(), closest_point_on_line(), closest_point_on_circle(), push_out_line()
- Notes: Two-pass algorithm: first line pass (walls), second line pass (previously hit), then point pass (corners)

### line_is_obstructed
- Signature: `bool line_is_obstructed(short polygon_index1, world_point2d *p1, short polygon_index2, world_point2d *p2, bool for_sounds=false)`
- Purpose: Test if a line-of-sight or sound path is blocked by solid walls or media boundaries
- Inputs: Start polygon/point, end polygon/point, optional sound obstruction flag
- Outputs/Return: true if obstructed
- Side effects: None (read-only query)
- Calls: find_line_crossed_leaving_polygon(), _find_line_crossed_leaving_polygon(), get_line_data(), find_adjacent_polygon(), get_polygon_data()
- Notes: Special handling for sound (can pass through transparent sides); detects media (water/lava) boundary crossings via separate check

### mark_map_collections
- Signature: `void mark_map_collections(bool loading)`
- Purpose: Scan map geometry and content to determine which texture/shape collections must be loaded
- Inputs: true = mark for loading; false = mark for unloading
- Outputs/Return: None
- Side effects: Calls mark_collection_for_loading/unloading() and mark_effect_collections()
- Calls: Scans all polygons, sides, media, and scenery objects for collection references
- Notes: Prevents unloading collections still in use

### turn_object_to_shit
- Signature: `void turn_object_to_shit(short garbage_object_index)`
- Purpose: Mark an object as garbage (dead body, spent ammunition) for cleanup
- Inputs: Object index
- Outputs/Return: None
- Side effects: Sets _object_is_garbage owner; may evict older garbage if per-polygon or per-map limit exceeded
- Calls: remove_map_object(), get_object_data(), get_polygon_data()
- Notes: Enforces MAXIMUM_GARBAGE_OBJECTS_PER_POLYGON and MAXIMUM_GARBAGE_OBJECTS_PER_MAP limits

### play_object_sound, play_polygon_sound, play_side_sound, play_world_sound
- Purpose: Enqueue sound playback from 3D world locations
- Inputs: Object/polygon/side/world indices, sound code, optional pitch/rewind flags
- Outputs/Return: None
- Side effects: Modifies sound system state via SoundManager
- Calls: get_object_data(), get_polygon_data(), get_side_data(), SoundManager::instance()->PlaySound(), get_object_sound_location()

### _sound_obstructed_proc
- Signature: `uint16 _sound_obstructed_proc(world_location3d *source, bool distinguish_obstruction_types)`
- Purpose: Determine how sound is affected by walls and media
- Inputs: Sound source location, distinction flag
- Outputs/Return: Flags (_sound_was_obstructed, _sound_was_media_obstructed, _sound_was_media_muffled)
- Side effects: None
- Calls: _sound_listener_proc(), line_is_obstructed(), get_polygon_data(), get_media_data()
- Notes: Detects three obstruction modes: walls, media boundary crossing, media muffling

## Control Flow Notes

**Initialization pipeline:**
- allocate_map_memory() ΓåÆ called once at engine startup
- initialize_map_for_new_game() ΓåÆ reset for fresh game start
- initialize_map_for_new_level() ΓåÆ called on level load (after geometry and saved objects are parsed elsewhere)
- mark_map_collections() ΓåÆ determine textures to load during level setup

**Per-frame entity updates (from marathon2.cpp):**
- update_world() calls move_projectiles(), move_monsters() which use translate_map_object()
- Sound propagation during animation frame: _sound_add_ambient_sources_proc() uses line_is_obstructed()
- Collision/obstruction checks during movement use keep_line_segment_out_of_walls()

**Object lifecycle:**
1. **Creation**: new_map_object*() ΓåÆ _new_map_object() ΓåÆ adds to polygon.first_object chain
2. **Movement**: translate_map_object() ΓåÆ remove from old polygon, add to new
3. **Cleanup**: remove_map_object() ΓåÆ unlink, mark slot free; or turn_object_to_shit() ΓåÆ mark as garbage, eventual collection

## External Dependencies
- Notable includes: map.h (API), FilmProfile.h (compat flags), interface.h (shapes/collections), monsters.h/projectiles.h/effects.h (entities), SoundManager.h (audio), InfoTree.h (XML parsing)
- Defined elsewhere: GetMemberWithBounds() (bounds helper), obj_clear/objlist_clear (memory), get_dynamic_limit() (resource limits), geometric helpers (closest_point_on_line, find_adjacent_polygon), entity accessors (get_monster_data, get_media_data)
- Lua integration: L_Invalidate_Object() called on deletion for script cleanup
