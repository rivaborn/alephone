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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `monster_data` | struct | 64-byte instance data storing type, vitality, position, target, flags, state, AI parameters |
| Monster types enum | enum | 40+ values (e.g., `_monster_marine`, `_monster_tick_energy`, `_monster_cyborg_major`) |
| Monster actions enum | enum | 12 values (`_monster_is_stationary`, `_monster_is_moving`, `_monster_is_attacking_close`, etc.) |
| Monster modes enum | enum | 5 values (`_monster_locked`, `_monster_losing_lock`, `_monster_lost_lock`, etc.) |
| Monster flags enum | enum | 6 flag bits (promoted, demoted, never activated, blind, deaf, teleport on deactivate) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MonsterList` | `vector<monster_data>` | global (extern) | Dynamic array of all active monsters in current level |
| `MAXIMUM_MONSTERS_PER_MAP` | macro ΓåÆ `get_dynamic_limit()` | global | Runtime-configurable limit on concurrent monsters |
| `LOCAL_INTERSECTING_MONSTER_BUFFER_SIZE` | macro ΓåÆ `get_dynamic_limit()` | global | Configurable local collision buffer size (default 16) |
| `GLOBAL_INTERSECTING_MONSTER_BUFFER_SIZE` | macro ΓåÆ `get_dynamic_limit()` | global | Configurable global collision buffer size (default 64) |

## Key Functions / Methods

### initialize_monsters
- **Signature:** `void initialize_monsters(void);`
- **Purpose:** Initialize monster system at engine startup.
- **Calls:** (via extern, likely called once at game init)

### initialize_monsters_for_new_level
- **Signature:** `void initialize_monsters_for_new_level(void);`
- **Purpose:** Reset monster state when a map loads; clears/reinitializes `MonsterList`.
- **Calls:** (per-level load)

### move_monsters
- **Signature:** `void move_monsters(void);`
- **Purpose:** Update all active monsters for one tick; advances animation, pathfinding, AI decision-making.
- **Notes:** Assumes constant tick duration (1 tick); called once per frame.

### new_monster / remove_monster
- **Signature:** `short new_monster(struct object_location *location, short monster_code);` / `void remove_monster(short monster_index);`
- **Purpose:** Spawn or despawn a monster; returns monster index on spawn.
- **Inputs:** location (world position + polygon), monster_code (type enum)
- **Outputs:** Monster index (short) on success; -1 or similar on failure (inferred)

### activate_monster / deactivate_monster
- **Signature:** `void activate_monster(short monster_index);` / `void deactivate_monster(short monster_index);`
- **Purpose:** Transition monster from inactive (no AI updates) to active (running AI loop), or vice versa.
- **Side effects:** Sets/clears `MONSTER_IS_ACTIVE` flag; affects frame-to-frame behavior.

### find_closest_appropriate_target
- **Signature:** `short find_closest_appropriate_target(short aggressor_index, bool full_circle);`
- **Purpose:** Find nearest hostile target for a monster (geometric or visibility-based).
- **Inputs:** aggressor index, flag for 360┬░ vs. directional search
- **Outputs:** Target monster/object index

### damage_monster / damage_monsters_in_radius
- **Signature:** `void damage_monster(short monster_index, short aggressor_index, short aggressor_type, world_point3d *epicenter, struct damage_definition *damage, short projectile_index);`
- **Purpose:** Apply damage to single monster or all monsters in radius; triggers hit animation, target change, death logic.
- **Inputs:** target, attacker, damage type, impact location, damage struct, projectile ID
- **Side effects:** Modifies monster vitality, action state, target lock; may trigger death or berserk mode

### change_monster_target
- **Signature:** `void change_monster_target(short monster_index, short target_index);`
- **Purpose:** Repoint monster's target; updates lock state and AI behavior.

### activate_nearby_monsters
- **Signature:** `void activate_nearby_monsters(short target_index, short caller_index, short flags, int32 max_range = -1);`
- **Purpose:** Activate monsters within range based on trigger flags (sound, line-of-sight, teleport, etc.).
- **Inputs:** trigger source, caller context, activation mode flags, optional range limit
- **Flags:** Control whether trigger crosses zone borders, passes through walls, respects activation biases, etc.

### legal_monster_move / legal_player_move
- **Signature:** `short legal_monster_move(short monster_index, angle facing, world_point3d *new_location);`
- **Purpose:** Validate proposed movement; check collision with walls, floors, ceilings.
- **Outputs:** Result code (success, blocked, out of bounds, etc.)

### get_monster_dimensions
- **Signature:** `void get_monster_dimensions(short monster_index, world_distance *radius, world_distance *height);`
- **Purpose:** Retrieve collision shape (radius and height) for a monster type.

### possible_intersecting_monsters
- **Signature:** `bool possible_intersecting_monsters(vector<short> *IntersectedObjectsPtr, unsigned maximum_object_count, short polygon_index, bool include_scenery);`
- **Purpose:** Query all monsters/objects in or near a polygon for collision checks.
- **Outputs:** Populates vector with object indices; used by `monsters_nearby(polygon_index)` macro.

### monster_moved / monster_died
- **Signature:** `void monster_moved(short target_index, short old_polygon_index);` / `void monster_died(short target_index);`
- **Purpose:** Update world state when monster changes polygon or dies; triggers death effects, target cleanup.

### Trivial helpers
- `get_monster_data()` ΓÇö Return pointer to `monster_data` for index (likely trivial inline/lookup).
- `get_monster_impact_effect()`, `get_monster_melee_impact_effect()` ΓÇö Fetch impact/sound effect for a monster type.
- `bump_monster()`, `accelerate_monster()` ΓÇö Simple physics/collision responses.
- `live_aliens_on_map()` ΓÇö Predicate for level-end condition.
- `monster_placement_index()`, `placement_index_to_monster_type()` ΓÇö Conversion between placement types and runtime types.

## Control Flow Notes
This file is a core part of the **per-frame update loop**. Typical sequence:
1. **Level load:** `initialize_monsters_for_new_level()` clears the list.
2. **Spawn/trigger:** `new_monster()`, `activate_nearby_monsters()` add/activate entities.
3. **Per tick:** `move_monsters()` updates all active monster AI, animation, position.
4. **Damage/interaction:** Combat and collision systems call `damage_monster()`, `legal_monster_move()`, callbacks like `monster_moved()`.
5. **Cleanup:** `remove_monster()` on death or despawn; `monster_died()` notifies systems.

Activation is lazyΓÇöinactive monsters do not update, reducing CPU cost in large maps.

## External Dependencies
- **`dynamic_limits.h`:** Runtime configuration of monster counts and buffer sizes.
- **`world.h`:** Spatial types (`world_point3d`, `world_distance`, `angle`, `world_location3d`), distance/angle math.
- **`<vector>`:** STL container for `MonsterList`.
- **`InfoTree`:** (forward declared) XML/config tree for MML parsing (`parse_mml_monsters()`, `parse_mml_damage_kicks()`).
- **`struct damage_definition`, `struct object_location`:** Defined elsewhere; used in function signatures.
