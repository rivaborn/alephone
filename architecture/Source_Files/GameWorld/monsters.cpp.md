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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `monster_data` | struct | Runtime state of single monster (vitality, position, mode, target, velocities) |
| `monster_definition` | struct | Static template defining monster properties (speed, attacks, visual range, sounds) |
| `damage_kick_definition` | struct | Velocity imparted to monsters by specific damage types; indexed by damage_type |
| `monster_pathfinding_data` | struct | Transient context passed to pathfinding cost function |
| `attack_definition` | struct | Melee/ranged attack properties (type, range, repetitions, offset, error) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MonsterList` / `monsters` | `vector<monster_data>` | global | All monsters on map; indexed by monster_index |
| `monster_definitions` | array | global (from monster_definitions.h) | Templates for each monster type |
| `damage_kick_definitions` | array | global | Velocity multipliers per damage type; MML-configurable |
| `original_damage_kick_definitions` | pointer | static | Backup for MML parsing; reset on reload |
| `monster_must_be_exterminated` | `vector<bool>` | static | Per-type flag for mission extermination requirement |
| `dynamic_world` | pointer | global | Current game state (tick_count, last_monster_to_get_time/path, civilian_count, etc.) |
| `static_world` | pointer | global | Static map data (environment_flags, mission_flags, etc.) |

## Key Functions / Methods

### move_monsters
- Signature: `void move_monsters(void)`
- Purpose: Main per-frame update loop for all monsters. Distributed AI: one monster gets pathfinding per frame, one gets target lock per frame to avoid CPU spike.
- Inputs: None (reads global `monsters`, `dynamic_world`, objects)
- Outputs/Return: None (mutates monster state, may deactivate/kill monsters)
- Side effects: Updates physics, animations, paths; may spawn effects/projectiles; modifies global counters; calls activate_nearby_monsters()
- Calls: `get_monster_definition()`, `animate_object()`, `cause_polygon_damage()`, `update_monster_vertical_physics_model()`, `generate_new_path_for_monster()`, `handle_moving_or_stationary_monster()`, `execute_monster_attack()`, `try_monster_attack()`, `set_monster_action()`, `kill_monster()`, `monster_died()`, deactivation helpers
- Notes: Loops through all monsters; state machine on action (waiting/moving/attacking_close/attacking_far/being_hit/dying_*). Handles animation keyframe events. Difficulty-level dependent civilian counter decay.

### try_monster_attack
- Signature: `static bool try_monster_attack(short monster_index)`
- Purpose: Validate if monster can attack target (LOS, range, obstruction); if valid, initiate attack animation.
- Inputs: `monster_index` (monster_data and target_index must be valid)
- Outputs/Return: `true` if attack initiated, `false` otherwise
- Side effects: Sets monster action, facing, attack_repetitions, ticks_since_attack; may play clear_sound if obstructed by ally
- Calls: `get_monster_data()`, `get_object_data()`, `get_monster_definition()`, `position_monster_projectile()`, `preflight_projectile()`, `line_is_obstructed()`, `get_monster_attitude()`, `play_object_sound()`
- Notes: Chooses melee vs ranged based on range; applies difficulty-level reduction to repetitions; random weapon choice if flagged; returns early if no valid target.

### execute_monster_attack
- Signature: `static void execute_monster_attack(short monster_index)`
- Purpose: Fire projectile(s) during attack keyframe event.
- Inputs: `monster_index` (must have valid target_index and be in attacking action)
- Outputs/Return: None
- Side effects: Calls `new_projectile()` 1ΓÇô2 times (twice if symmetric fire); updates attack.dy
- Calls: `get_monster_data()`, `get_monster_definition()`, `get_object_data()`, `position_monster_projectile()`, `new_projectile()`
- Notes: Aborts if target becomes invalid mid-animation. Symmetric weapons flip dy, fire, flip back.

### new_monster
- Signature: `short new_monster(struct object_location *location, short monster_type)`
- Purpose: Spawn a new monster on the map; apply difficulty-based promotion/demotion/dropping.
- Inputs: `location` (polygon, coordinates, flags), `monster_type`
- Outputs/Return: Monster index if successful, `NONE` otherwise
- Side effects: Allocates monster_data slot; creates map object; tracks civilian count; increments object frequency counters; may demote/promote type
- Calls: `get_monster_definition()`, `new_map_object()`, `get_object_data()`, `nearest_goal_polygon_index()`, `object_was_just_added()`
- Notes: Difficulty level affects drop rates (wuss/easy skip monsters), major/minor promotion/demotion. Initializes vitality=NONE (lazy init on activation). Handles blind/deaf/float flags from location.

### activate_nearby_monsters
- Signature: `void activate_nearby_monsters(short target_index, short caller_index, short flags, int32 max_range)`
- Purpose: Flood-fill activation from caller's polygon; activate dormant monsters and lock onto target.
- Inputs: `target_index` (lock target, or NONE), `caller_index` (origin), `flags` (_pass_one_zone_border, _activate_invisible_monsters, etc.), `max_range` (optional cost cap)
- Outputs/Return: None
- Side effects: Activates inactive monsters; changes targets; calls `activate_monster()`, `change_monster_target()`, `find_closest_appropriate_target()`
- Calls: `get_monster_data()`, `get_object_data()`, `flood_map()`, `monster_activation_flood_proc()`, `get_polygon_data()`, `get_monster_attitude()`, `switch_target_check()`, `activate_monster()`
- Notes: Defers `find_closest_appropriate_target()` calls to avoid reentrancy; respects deaf/invisible flags and activation biases.

### monster_pathfinding_cost_function
- Signature: `int32 monster_pathfinding_cost_function(short source_polygon_index, short line_index, short destination_polygon_index, void *vdata)`
- Purpose: Cost evaluator for A* pathfinding; returns -1 to block path, positive cost otherwise.
- Inputs: `vdata` points to `monster_pathfinding_data` (definition, monster, cross_zone_boundaries)
- Outputs/Return: Cost (negative = impassable)
- Side effects: None
- Calls: `get_polygon_data()`, `get_line_data()`, `get_object_data()`, `monster_can_enter_platform()`, `monster_can_leave_platform()`, `find_flooding_polygon()`, `get_media_data()`, `get_platform_data()`, `IsMediaDangerous()`
- Notes: Checks solid lines, platform accessibility, height changes, line width, media (water/lava/goo). Blocks dangerous media for non-flying/floating. Counts monsters in destination (overcrowding penalty).

### position_monster_projectile
- Signature: `static short position_monster_projectile(short aggressor_index, short target_index, struct attack_definition *attack, world_point3d *origin, world_point3d *destination, world_point3d *_vector, angle theta)`
- Purpose: Compute projectile spawn point and direction vector, accounting for attack offsets (dx, dy, dz).
- Inputs: Aggressor, target, attack definition, origin (output), destination (optional; NULL = fire along facing), theta (facing)
- Outputs/Return: Polygon index at final origin; updates origin, optionally destination and elevation
- Side effects: None (pure calculation)
- Calls: `get_monster_data()`, `get_object_data()`, `get_monster_dimensions()`, `translate_point2d()`, `find_new_object_polygon()`
- Notes: If destination provided, shoots at 3/4 height on target. Otherwise uses monster's elevation angle.

### damage_monsters_in_radius / damage_monster
- Signature: `void damage_monsters_in_radius(short primary_target_index, short aggressor_index, short aggressor_type, world_point3d *epicenter, short epicenter_polygon_index, world_distance radius, struct damage_definition *damage, short projectile_index)`
- Purpose: Apply splash damage in radius; find all monsters in range and call `damage_monster()` on each.
- Inputs: Damage definition, epicenter, radius, aggressor context
- Outputs/Return: None
- Side effects: Applies damage; may change targets; may trigger death; updates vitality; applies damage kicks
- Calls: `possible_intersecting_monsters()`, `damage_monster()` for each in radius

### kill_monster
- Signature: `static void kill_monster(short monster_index)`
- Purpose: Handle death animation completion; remove monster from map; clean up references.
- Inputs: `monster_index` (monster must be dying)
- Outputs/Return: None
- Side effects: Marks slot as free; removes object; calls `monster_died()`; triggers `L_Invalidate_Monster()` for Lua
- Calls: `get_monster_data()`, `get_object_data()`, `remove_map_object()`, `monster_died()`, `L_Invalidate_Monster()`

### Serialization Functions
- `unpack_monster_data()` / `pack_monster_data()`: Stream Γåö monster_data array
- `unpack_monster_definition()` / `pack_monster_definition()`: Stream Γåö monster_definition array
- `unpack_m1_monster_definition()`: Marathon 1 format ΓåÆ current format (adds compatibility flags)

### MML Parsing
- `parse_mml_damage_kicks()`: Parse XML for damage kick customization; backs up originals
- `reset_mml_damage_kicks()`: Restore original values
- `parse_mml_monsters()`: Parse XML for monster flags (must_be_exterminated)

## Control Flow Notes
Integrated into main game loop via `move_monsters()` (called once per frame/tick):
1. **Update phase**: Physics, animation, polygon damage
2. **AI phase** (time-shared):
   - One unlocked/lost-lock monster gets target lock
   - One monster needing path gets pathfinding
3. **Action phase**: Based on monster.action state machine:
   - Stationary/moving: call `handle_moving_or_stationary_monster()` (movement, pathfinding, obstruction handling)
   - Attacking: execute on keyframe, check animation flags for completion
   - Being hit: resolve external velocity, return to moving
   - Dying: apply physics, trigger shrapnel at keyframe, call `kill_monster()` at last frame
4. **Cleanup**: Inactive monsters scan for targets passively; reset frame counters

Monsters have distributed AI to avoid per-frame overhead: pathfinding and targeting are round-robined across the monster list per frame.

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
