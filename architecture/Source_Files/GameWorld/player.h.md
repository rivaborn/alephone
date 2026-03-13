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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `player_settings_definition` | struct | Configuration: initial energy/oxygen, powerup values, luminosity, vision parameters, oxygen drain rates |
| `physics_variables` | struct | Per-frame movement state: position, velocity, direction, elevation, angular velocity, floor/ceiling heights, step animation phase |
| `player_data` | struct | Complete player state: identifier, location, weapon/item inventory, suit energy, powerups, damage records, control panel state, teleport state, reincarnation delay |
| `damage_record` | struct | Cumulative damage dealt/taken: total damage (int32) and kill count (int16) |
| `player_shape_definitions` | struct | Visual model indices: collection, death frames, leg/torso animations for different actions/weapons |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `player_settings` | `player_settings_definition` | global | Current player configuration (loaded from MML); accessed via macro `NATURAL_LIGHT_INTENSITY` |
| `players` | `player_data*` | global | Dynamic array of all players in game |
| `local_player_index` / `local_player` | int16 / `player_data*` | global | Index and pointer to the human player on this machine |
| `current_player_index` / `current_player` | int16 / `player_data*` | global | Index and pointer to the player being updated/rendered (may differ from local in networked scenarios) |
| `team_damage_given[]` / `team_damage_taken[]` | `damage_record[]` | global | Per-team cumulative damage statistics |
| `team_monster_damage_*[]` / `team_friendly_fire[]` | `damage_record[]` | global | Per-team damage from/to monsters and friendly fire |

## Key Functions / Methods

### initialize_players
- **Signature:** `void initialize_players(void)`
- **Purpose:** One-time setup of player system at game startup
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates memory, initializes global player arrays
- **Calls:** `allocate_player_memory()`, likely player-related setup
- **Notes:** Called during game initialization phase

### allocate_player_memory
- **Signature:** `void allocate_player_memory(void)`
- **Purpose:** Dynamically allocate player array memory (respects `MAXIMUM_NUMBER_OF_PLAYERS` which varies by DEMO mode)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates `players` global array
- **Calls:** Memory allocation routines
- **Notes:** May be called multiple times to resize

### new_player
- **Signature:** `short new_player(short team, short color, short player_identifier, new_player_flags flags)`
- **Purpose:** Create and initialize a new player in the game
- **Inputs:** Team index, color index, unique player identifier, flags (`new_player_make_local`, `new_player_make_current`)
- **Outputs/Return:** Player array index (short), or error code
- **Side effects:** Modifies `local_player_index`, `current_player_index`; initializes weapon/physics state
- **Calls:** `initialize_player_weapons()`, `initialize_player_physics_variables()`
- **Notes:** Flags determine if this player becomes local or current

### update_players
- **Signature:** `void update_players(ActionQueues* inActionQueuesToUse, bool inPredictive)`
- **Purpose:** Main per-frame update for all players (physics, weapons, state changes)
- **Inputs:** Action queue pointer (input state), predictive flag (for state rollback)
- **Outputs/Return:** None
- **Side effects:** Updates all player physics, position, animation, weapon state; modifies `local_player`, `current_player`
- **Calls:** `update_player_physics_variables()`, `update_player_weapons()`, weapon/powerup handlers
- **Notes:** Assumes 1 tick elapsed; used for networked prediction

### damage_player
- **Signature:** `void damage_player(short monster_index, short aggressor_index, short aggressor_type, struct damage_definition *damage, short projectile_index)`
- **Purpose:** Apply damage to a player from a source (monster, projectile, environment)
- **Inputs:** Monster index (source), aggressor player/team, aggressor type, damage definition, projectile identifier
- **Outputs/Return:** None
- **Side effects:** Reduces player suit energy; updates damage records; may kill/zombify player
- **Calls:** Likely calls powerup handlers, death state setters
- **Notes:** Invoked by collision/hit detection systems

### update_player_physics_variables
- **Signature:** `void update_player_physics_variables(short player_index, uint32 action_flags, bool predictive)`
- **Purpose:** Simulate physics for one player (movement, collision, gravity, floor detection)
- **Inputs:** Player index, action flags encoding movement intent, predictive mode flag
- **Outputs/Return:** None
- **Side effects:** Updates `player.variables` (position, velocity, direction, floor/ceiling height); moves player in world
- **Calls:** Collision/floor-finding routines from `map.h`
- **Notes:** Core physics engine; separate from visual animation

### process_aim_input
- **Signature:** `uint32 process_aim_input(uint32 action_flags, fixed_yaw_pitch delta)`
- **Purpose:** Integrate high-precision aiming input (e.g., mouse deltas) into action flags
- **Inputs:** Current action flags, yaw/pitch delta from aim input
- **Outputs/Return:** Updated action flags with yaw/pitch encoded
- **Side effects:** Updates virtual aim state
- **Calls:** None visible
- **Notes:** Handles precision aiming separate from discrete input; used for smooth mouse/gamepad aim

### set_local_player_index / set_current_player_index
- **Signature:** `void set_local_player_index(short); void set_current_player_index(short)`
- **Purpose:** Change which player is local (human-controlled on this machine) or current (being updated)
- **Inputs:** New player index
- **Outputs/Return:** None
- **Side effects:** Updates global `local_player_index`, `current_player_index` and their pointers
- **Calls:** None visible
- **Notes:** Must use these setters, not direct assignment (per comment)

---

## Control Flow Notes
- **Initialization phase:** `initialize_players()` ΓåÆ `allocate_player_memory()`, then per player `new_player()` ΓåÆ `initialize_player_weapons()`, `initialize_player_physics_variables()`
- **Per-frame update:** `update_players(ActionQueues*, bool)` called from main game loop; updates all physics, weapons, damage, powerups
- **Damage/death flow:** Collision detection invokes `damage_player()` ΓåÆ powerup effects applied ΓåÆ possible state transitions (`SET_PLAYER_DEAD_STATUS()`, `SET_PLAYER_ZOMBIE_STATUS()`)
- **Physics simulation** happens independently in physics.c functions; player.h declares interfaces only

## External Dependencies
- **Notable includes:** `cseries.h` (core types, macros), `world.h` (coordinate types, angle, world_point3d), `map.h` (world geometry, WORLD_ONE constant), `weapons.h` (weapon type enums)
- **Defined elsewhere:**
  - `ActionQueues`, `ModifiableActionQueues` (forward declarations; defined elsewhere)
  - `InfoTree` (forward declaration; XML parsing support)
  - All `unpack_*` / `pack_*` functions (implement serialization, declared here)
  - Weapon and physics constants from included headers
