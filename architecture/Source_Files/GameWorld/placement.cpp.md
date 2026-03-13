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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `object_frequency_definition` | struct | Defines spawn rules (initial, min, max, random counts and chance) |
| `object_location` | struct | 3D position + polygon index + facing angle |
| `world_point2d` | typedef | 2D world coordinates |
| `polygon_data` | struct | Map polygon (from map.h); checked for object placement validity |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `object_placement_info[2*MAXIMUM_OBJECT_TYPES]` | `object_frequency_definition[]` | static | Combined monster+item placement rules; saved with game state |
| `monster_placement_info` | pointer | static | Points into `object_placement_info` for monsters (offset by MAXIMUM_OBJECT_TYPES) |
| `item_placement_info` | pointer | static | Points into `object_placement_info` for items |
| `delay` | `int32` (in `recreate_objects`) | static | Tick counter for spawn interval timing |

## Key Functions / Methods

### load_placement_data
- **Signature:** `void load_placement_data(uint8 *_monsters, uint8 *_items)`
- **Purpose:** Parse placement rules from WAD file streams and initialize global placement tables.
- **Inputs:** Two byte streams containing packed monster and item placement definitions.
- **Outputs/Return:** None; populates `monster_placement_info` and `item_placement_info`.
- **Side effects:** Clears global arrays; unpacks data from binary streams; validates placement data in DEBUG mode; forces marine (player monster) to never spawn.
- **Calls:** `objlist_clear()`, `unpack_object_frequency_definition()`, `obj_clear()`, `dprintf()` (debug).
- **Notes:** Includes disabled code (if 0) that would filter out network-only items in single-player. Marine (index 0) is always cleared to prevent duplicate player spawns.

### place_initial_objects
- **Signature:** `void place_initial_objects(void)`
- **Purpose:** Spawn initial objects (monsters and items) as configured by placement rules at level start.
- **Inputs:** None; reads from `monster_placement_info`, `item_placement_info`, game options.
- **Outputs/Return:** None; creates monsters and items via `add_objects()`.
- **Side effects:** Modifies `dynamic_world->current_monster_count`, `current_item_count`, `random_monsters_left`, `random_items_left`. Initializes civilian casualty counters.
- **Calls:** `add_objects()` for each type with initial_count > 0. Respects film_profile.initial_monster_fix and `_monsters_replenish` option.
- **Notes:** Loops start at index 1 for monsters (skipping marine at 0) but index 0 for items.

### recreate_objects
- **Signature:** `void recreate_objects(void)`
- **Purpose:** Periodically check and respawn monsters/items to maintain minimum counts or add random spawns (called each frame).
- **Inputs:** None; reads `dynamic_world->tick_count`, game options, placement rules.
- **Outputs/Return:** None; calls `_recreate_objects()` if spawn interval elapsed.
- **Side effects:** Updates static `delay` counter; may create new objects via `_recreate_objects()`.
- **Calls:** `_recreate_objects()` (conditional, ~every 15 ticks for monsters if replenishment enabled; always for items).
- **Notes:** Detects time-backwards (new game started) and resets delay. Only respawns if `_monsters_replenish` flag is set.

### object_was_just_added / object_was_just_destroyed
- **Signature:** 
  - `void object_was_just_added(short object_class, short object_type)`
  - `void object_was_just_destroyed(short object_class, short object_type)`
- **Purpose:** Increment/decrement live object counters; trigger respawn if below minimum.
- **Inputs:** Object class (_object_is_monster or _object_is_item) and type index.
- **Outputs/Return:** None; modifies `dynamic_world->current_monster_count[]` or `current_item_count[]`.
- **Side effects:** Decrements counter; if below minimum, calls `add_objects(..., 1, false)` to respawn one object.
- **Calls:** `add_objects()` (only in destroyed).
- **Notes:** Destroyed handles held items gracefully (may have count=0 even if object exists). Respawn respects minimum_count from placement rules.

### get_random_player_starting_location_and_facing
- **Signature:** `short get_random_player_starting_location_and_facing(short max_player_index, short team, struct object_location *location)`
- **Purpose:** Find a safe/favorable spawn point for a player, weighted by distance from monsters and other players.
- **Inputs:** Current max player index, team (or NONE), output location struct.
- **Outputs/Return:** Starting location index; fills `*location` with coordinates and facing.
- **Side effects:** Calls visibility queries (point_is_player_visible, point_is_monster_visible).
- **Calls:** `get_player_starting_location_and_facing()` (map.c), `point_is_player_visible()`, `point_is_monster_visible()` (map.c).
- **Notes:** Falls back to team=NONE if no team-specific starts exist. Weighted metric: `player_distance + (monster_distance >> 1)` prefers distance from players slightly more. Fallback to last candidate if all locations have monsters/players exactly on them (extremely unlikely edge case).

### _recreate_objects (private)
- **Signature:** `static void _recreate_objects(short object_type, short max_object_types, struct object_frequency_definition *placement_info, short *object_counts, short *random_counts)`
- **Purpose:** Internal loop to check and spawn objects to maintain min/max/random spawn rules.
- **Inputs:** Object class, count of types, placement rules array, current counts array, remaining random spawns array.
- **Outputs/Return:** None; calls `add_objects()` for objects needing respawn.
- **Side effects:** Decrements `random_counts[]` if random spawn is consumed.
- **Calls:** `add_objects()` (if objects_to_add > 0).
- **Notes:** Respects minimum_count, maximum_count, and random_chance. Random spawn consumed only if random_count != NONE (NONE means infinite). Monsters skip index 0 (marine).

### add_objects (private)
- **Signature:** `static void add_objects(short object_class, short object_type, short count, bool is_initial_drop)`
- **Purpose:** Create `count` instances of an object type, finding valid locations and calling creation functions.
- **Inputs:** Object class, type, count, initial_drop flag.
- **Outputs/Return:** None; creates objects.
- **Side effects:** Creates monster/item objects; may activate monsters; updates paths and targets for non-initial spawns.
- **Calls:** `pick_random_initial_location_of_type()`, `choose_invisible_random_point()`, `pick_random_facing()`, `new_item()`, `new_monster()`, `activate_monster()`, `find_closest_appropriate_target()`.
- **Notes:** Respects _reappears_in_random_location flag (initial spots vs. random spawns). Newly spawned non-initial monsters are activated and assigned targets. Silently skips if location cannot be found.

### polygon_is_valid_for_object_drop (private)
- **Signature:** `static bool polygon_is_valid_for_object_drop(world_point2d *location, short polygon_index, short object_type, bool initial_drop, bool is_random_location)`
- **Purpose:** Validate whether a polygon can accept a spawned object (checks type, detachment, visibility, collisions).
- **Inputs:** 2D location, polygon index, object type, initial vs. respawn flag, random vs. predefined flag.
- **Outputs/Return:** true if valid for placement; false otherwise.
- **Side effects:** None (query only); calls visibility check.
- **Calls:** `get_polygon_data()`, `point_is_player_visible()`, `get_object_data()`.
- **Notes:** Rejects impassable/teleporter/platform polygons for random spawns. For predefined initial locations, allows some polygon types if not random. Rejects if player visible (unless initial_drop). For random spawns, checks for colliding items (items) or projectiles/monsters/effects (monsters). For predefined locations, checks exact position matching.

### pick_random_initial_location_of_type (private)
- **Signature:** `static bool pick_random_initial_location_of_type(short saved_type, short type, struct object_location *location)`
- **Purpose:** Select a random predefined spawn location from the map for a given object type.
- **Inputs:** Saved object type (_saved_item or _saved_monster), object type index, output location.
- **Outputs/Return:** true if found; fills `*location`.
- **Side effects:** None.
- **Calls:** `polygon_is_valid_for_object_drop()`.
- **Notes:** Iterates saved_objects (predefined placements from map editor) starting at random offset. Validates each candidate. Returns first valid match.

### choose_invisible_random_point (private)
- **Signature:** `static bool choose_invisible_random_point(short *polygon_index, world_point2d *p, short object_type, bool initial_drop)`
- **Purpose:** Find a random valid polygon for invisible (non-predefined) object spawning.
- **Inputs:** Output polygon index, output 2D point, object type, initial_drop flag.
- **Outputs/Return:** true if found; fills outputs.
- **Side effects:** None.
- **Calls:** `find_center_of_polygon()`, `polygon_is_valid_for_object_drop()`.
- **Notes:** Retries up to INVISIBLE_RANDOM_POINT_RETRIES (10) times. Places object at polygon center.

### pick_random_facing (private)
- **Signature:** `static short pick_random_facing(short polygon_index, world_point2d *location)`
- **Purpose:** Generate a facing angle for a monster that preferentially points away from walls.
- **Inputs:** Polygon index, 2D location.
- **Outputs/Return:** Facing angle (0ΓÇô2879).
- **Side effects:** None.
- **Calls:** `find_new_object_polygon()`.
- **Notes:** Tests 4 cardinal directions (QUARTER_CIRCLE increments). Returns first direction that crosses into an adjacent polygon (points "outward"). Falls back to random direction if all test directions hit walls.

## Control Flow Notes
- **Level Load:** `load_placement_data()` called once to initialize global tables.
- **Level Start:** `place_initial_objects()` called to spawn initial monsters/items.
- **Per Frame:** `recreate_objects()` called from update loop to respawn/maintain counts.
- **Object Lifecycle:** `object_was_just_added()` on creation, `object_was_just_destroyed()` on removal; destroyed may trigger respawn.

## External Dependencies
- **map.h:** Polygon/map geometry (polygon_data, object_location, world_point2d/3d), map queries (find_new_object_polygon, point_in_polygon, find_center_of_polygon), object creation (new_map_object), visibility checks (point_is_player_visible, point_is_monster_visible, get_polygon_data, get_object_data).
- **monsters.h:** Monster creation/activation (new_monster, activate_monster, find_closest_appropriate_target), loading (mark_monster_collections, load_monster_sounds).
- **items.h:** Item creation (new_item).
- **FilmProfile.h:** Compatibility flag `film_profile.initial_monster_fix` (controls whether initial monsters spawn based on film mode).
- **cseries.h:** Utility (cseries/csmacros.h for obj_clear, objlist_clear, TEST_FLAG, etc.).
