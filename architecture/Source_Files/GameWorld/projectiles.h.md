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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `projectile_data` | struct | 32-byte record: type, owner, target, position, velocity, damage, state flags |
| Projectile type enum | enum | 40+ types: `_projectile_rocket`, `_projectile_grenade`, `_projectile_smg_bullet`, etc. |
| `translate_projectile()` flags | enum | Result codes: `_flyby_of_current_player`, `_projectile_hit`, `_projectile_hit_monster`, etc. |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ProjectileList` | `std::vector<projectile_data>` | global | Active projectiles in the current map |
| `MAXIMUM_PROJECTILES_PER_MAP` | macro | global | Dynamic limit, retrieved from resource/config |

## Key Functions / Methods

### new_projectile
- Signature: `short new_projectile(world_point3d *origin, short polygon_index, world_point3d *_vector, angle delta_theta, short type, short owner_index, short owner_type, short intended_target_index, _fixed damage_scale)`
- Purpose: Create a new projectile in the world
- Inputs: Origin point, polygon, velocity vector, angle offset, type, owner, target, damage scale
- Outputs/Return: Projectile index (slot in ProjectileList), or NONE if max reached
- Side effects: Allocates a slot in ProjectileList; may trigger sounds/visuals via mark_projectile_collections
- Calls: Likely uses slot management, mark_projectile_collections, load_projectile_sounds
- Notes: Must call preflight_projectile first to verify the projectile can legally exist at that location

### preflight_projectile
- Signature: `bool preflight_projectile(world_point3d *origin, short polygon_index, world_point3d *_vector, angle delta_theta, short type, short owner, short owner_type, short *target_index)`
- Purpose: Test whether a projectile can be created without actually creating it
- Inputs: Same as new_projectile
- Outputs/Return: Boolean (valid or invalid); writes corrected target_index if guided
- Side effects: None
- Calls: Collision/validation routines (not visible in header)
- Notes: Used to validate weapon fire before instantiation

### translate_projectile
- Signature: `uint16 translate_projectile(short type, world_point3d *old_location, short old_polygon_index, world_point3d *new_location, short *new_polygon_index, short owner_index, short *obstruction_index, short *last_line_index, bool preflight, short projectile_indexx)`
- Purpose: Move a projectile and detect collisions; returns flags indicating what was hit
- Inputs: Projectile type, old/new positions, polygons, owner, preflight flag, projectile index
- Outputs/Return: Bitmask of collision flags (hit, hit_monster, hit_floor, hit_media, etc.); obstruction details via output pointers
- Side effects: May call detonation logic if collision detected; modifies new_location if penetrating media
- Calls: Collision detection, detonation (deferred via flags)
- Notes: Position may differ from old_location due to media-boundary penetration; caller must handle collision flags

### move_projectiles
- Signature: `void move_projectiles(void)`
- Purpose: Advance all active projectiles by one game tick
- Inputs: None (operates on global ProjectileList)
- Outputs/Return: None
- Side effects: Updates position, velocity, contrail counters for all projectiles; calls translate_projectile and handles collisions; may remove or detonate projectiles
- Calls: translate_projectile, detonation, remove_projectile
- Notes: Assumes ╬öt = 1 tick; called once per frame

### remove_projectile / remove_all_projectiles
- Signature: `void remove_projectile(short projectile_index)` / `void remove_all_projectiles(void)`
- Purpose: Delete a projectile or clear all projectiles from the world
- Outputs/Return: None
- Side effects: Deallocates slot in ProjectileList; may trigger cleanup sounds
- Calls: Slot management, mark_projectile_collections
- Notes: remove_all_projectiles used on map transition or level restart

### detonation-related
- `void detonate_projectile(world_point3d *origin, short polygon_index, short type, short owner_index, short owner_type, _fixed damage_scale)` ΓÇö Explode a projectile at a location, handling impact damage and effects

### Accessor / utility
- `bool ProjectileIsGuided(short Type)` ΓÇö Check if a projectile type auto-seeks targets
- `projectile_data *get_projectile_data(const short projectile_index)` ΓÇö Retrieve data for a projectile by index
- Packing/unpacking: `unpack_projectile_data`, `pack_projectile_data`, `unpack_projectile_definition`, etc. ΓÇö Serialization for save/load and network sync

## Control Flow Notes
**Typical lifecycle within frame loop:**
1. **Weapon fire**: `preflight_projectile()` validates; `new_projectile()` creates and inserts into ProjectileList
2. **Each tick**: `move_projectiles()` iterates all active projectiles, calls `translate_projectile()` to detect collisions
3. **On collision**: `translate_projectile()` returns flags; `detonation_projectile()` or `remove_projectile()` cleans up
4. **Owner death**: `orphan_projectiles()` handles dangling references when a monster/player dies
5. **Level transition**: `remove_all_projectiles()` clears state

## External Dependencies
- **dynamic_limits.h**: `get_dynamic_limit(_dynamic_limit_projectiles)` to retrieve max projectile count
- **world.h**: angle, world_point3d, world_distance, _fixed types; coordinate math and rotation
- **&lt;vector&gt;**: STL vector for dynamic projectile list
- **Presumed elsewhere**: Object system (object_index), polygon/polygon system (polygon_index), collision/physics, effects/sounds, monster types
