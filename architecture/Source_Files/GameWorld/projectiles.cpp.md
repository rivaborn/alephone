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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `projectile_data` | struct | Instance data for an active projectile (location, owner, target, flags, physics) |
| `projectile_definition` | struct | Template definition for a projectile type (speed, radius, effects, damage, flags) |
| `damage_definition` | struct | Damage parameters (type, base amount, random variance, scale) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `IntersectedObjects` | `vector<short>` | static | Growable list of object indices for collision checking within current polygon |
| `projectiles` | `projectile_data*` | global | Array of all active projectiles (from external vector) |
| `projectile_definitions` | `projectile_definition*` | global | Array of projectile type definitions (imported) |
| `alien_projectile_override` | `short` | global | Override projectile type for alien weapons (copy-protection) |
| `human_projectile_override` | `short` | global | Override projectile type for human weapons (copy-protection) |

## Key Functions / Methods

### new_projectile
- Signature: `short new_projectile(world_point3d *origin, short polygon_index, world_point3d *_vector, angle delta_theta, short type, short owner_index, short owner_type, short intended_target_index, _fixed damage_scale)`
- Purpose: Create a new projectile instance and place it in the game world
- Inputs: Origin position/polygon, velocity vector, angle error, type, owner identity, intended target, damage scaling
- Outputs/Return: Projectile index, or NONE if creation failed
- Side effects: Allocates projectile slot, creates map object, initializes physics state
- Calls: `adjust_projectile_type`, `get_projectile_definition`, `new_map_object3d`, `normalize_angle`, `L_Call_Projectile_Created`
- Notes: Applies random angle/elevation error based on delta_theta; selects overridden type if applicable; handles media projectile promotion

### move_projectiles
- Signature: `void move_projectiles(void)`
- Purpose: Update all active projectiles for one game tick
- Inputs: None (reads global projectiles array)
- Outputs/Return: None (modifies projectile state and game world)
- Side effects: Moves projectiles, applies damage, creates effects, removes expired projectiles, plays sounds
- Calls: `animate_object`, `update_guided_projectile`, `translate_projectile`, `damage_monsters_in_radius`, `damage_monster`, `damage_scenery`, `new_effect`, `play_object_sound`, `remove_projectile`, `try_and_add_player_item`, `L_Call_Projectile_Detonated`, `track_contrail_interpolation`
- Notes: Applies gravity (half/full/double based on flags); skips collision tests if projectile is invisible; handles penetrates-media-boundary flag; clamps speed for difficulty levels

### translate_projectile
- Signature: `uint16 translate_projectile(short type, world_point3d *old_location, short old_polygon_index, world_point3d *new_location, short *new_polygon_index, short owner_index, short *obstruction_index, short *last_line_index, bool preflight, short projectile_index)`
- Purpose: Perform collision detection and report what a projectile hit as it moves
- Inputs: Projectile type, old/new positions, owner, preflight flag
- Outputs/Return: Flags indicating what was hit; modifies new_location to collision point, sets obstruction_index and new_polygon_index
- Side effects: Clears and populates IntersectedObjects list
- Calls: `get_projectile_definition`, `get_polygon_data`, `get_media_data`, `possible_intersecting_monsters`, `find_line_crossed_leaving_polygon`, `find_line_intersection`, `find_adjacent_polygon`, `find_floor_or_ceiling_intersection`, `get_object_data`, `get_monster_dimensions`, `get_scenery_dimensions`, `distance2d`, `try_and_toggle_control_panel`
- Notes: Checks landscape/ceiling/floor/media/monsters/scenery in order; handles transparent lines and media boundary penetration; uses "traveled_underneath" flag for PMB logic

### remove_projectile
- Signature: `void remove_projectile(short projectile_index)`
- Purpose: Destroy a projectile and clean up its resources
- Inputs: Projectile index
- Outputs/Return: None
- Side effects: Removes map object, marks projectile slot as free, invalidates Lua references
- Calls: `get_projectile_data`, `L_Invalidate_Projectile`, `remove_map_object`

### update_guided_projectile
- Signature: `static void update_guided_projectile(short projectile_index)`
- Purpose: Adjust trajectory of guided projectiles toward target
- Inputs: Projectile index
- Outputs/Return: None
- Side effects: Modifies projectile facing and elevation
- Calls: `get_projectile_data`, `get_monster_data`, `get_object_data`, `get_monster_dimensions`
- Notes: Limits turn rate based on difficulty; breaks lock if target is invisible (except on total carnage); uses long-distance friendly math to prevent overflow

### detonate_projectile
- Signature: `void detonate_projectile(world_point3d *origin, short polygon_index, short type, short owner_index, short owner_type, _fixed damage_scale)`
- Purpose: Apply area-of-effect damage at a location
- Inputs: Detonation position, projectile type, owner, damage scale
- Outputs/Return: None
- Side effects: Damages monsters in radius, creates detonation effect, fires Lua callback
- Calls: `get_projectile_definition`, `damage_monsters_in_radius`, `new_effect`, `L_Call_Projectile_Detonated`

### preflight_projectile
- Signature: `bool preflight_projectile(world_point3d *origin, short origin_polygon_index, world_point3d *destination, angle delta_theta, short type, short owner, short owner_type, short *obstruction_index)`
- Purpose: Validate that a projectile can legally be fired from the given location
- Inputs: Origin, trajectory, projectile type, owner
- Outputs/Return: true if legal, false if in floor/ceiling/outside map; sets obstruction_index to first monster hit
- Side effects: None (aside from output parameters)
- Calls: `get_projectile_definition`, `get_polygon_data`, `get_media_data`, `translate_projectile`
- Notes: Used by weapon code before firing; ensures elevation not too extreme and position within valid bounds

## Control Flow Notes
**Frame/Update Cycle**: `move_projectiles` is called once per game tick (30 ticks/sec). Within each tick:
1. Animate projectile shape
2. Check if animation should stop projectile
3. Apply gravity based on flags (half/full/double affected)
4. Update guided projectile targeting (every other tick)
5. Call `translate_projectile` to detect collisions
6. If hit detected: apply damage, play sounds, create effects, possibly remove projectile
7. If no hit (or penetrating media): move to new location, create contrail effects, check max range

**Initialization**: `new_projectile` creates a projectile before it enters the frame loop.
**Cleanup**: Projectiles are removed by `move_projectiles` (on collision/expiry), `remove_projectile` (explicit), or `remove_all_projectiles` (level cleanup).

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
