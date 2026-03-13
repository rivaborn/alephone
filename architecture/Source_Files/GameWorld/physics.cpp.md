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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `physics_variables` | struct | Player's dynamic physics state (position, velocity, direction, elevation, flags) |
| `physics_constants` | struct | Model parameters: accelerations, velocities, deceleration rates, dimensions |
| `fixed_yaw_pitch` | struct | High-precision aim delta (yaw and pitch components) |
| `action_flags` | uint32 | Packed input state: movement, aiming, weapons, modifiers |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `physics_models` | `physics_constants[NUMBER_OF_PHYSICS_MODELS]` | static | Physics parameters for each model (walking/running variants) |
| `vir_aim_delta` | `fixed_yaw_pitch` | static | Virtual aim offset for local player; bridges high-precision input to low-precision network encoding |
| `saved_points`, `saved_thetas` | pointers (DEBUG) | static | Divergence checking data for netplay debugging |

## Key Functions / Methods

### initialize_player_physics_variables
- **Signature:** `void initialize_player_physics_variables(short player_index)`
- **Purpose:** Set up physics state when player spawns or level loads
- **Inputs:** Player index
- **Outputs/Return:** None; modifies player's `variables` and monster/object records
- **Side effects:** Writes to player data, initializes direction/position from object data, calls `instantiate_physics_variables()`
- **Calls:** `resync_virtual_aim()`, `get_physics_constants_for_model()`, `instantiate_physics_variables()`, `get_player_data()`, `get_monster_data()`, `get_object_data()`
- **Notes:** Zeros physics_variables except for position (copied from object), direction, elevation; called once at level start

### update_player_physics_variables
- **Signature:** `void update_player_physics_variables(short player_index, uint32 action_flags, bool predictive)`
- **Purpose:** Main per-frame physics update entry point
- **Inputs:** Player index, action flags (movement/aiming), predictive mode flag
- **Outputs/Return:** None; modifies player physics and world state
- **Side effects:** Updates player position/velocity, checks collisions, moves objects, applies gravity
- **Calls:** `physics_update()`, `instantiate_physics_variables()`
- **Notes:** Predictive=true reduces state changes for netplay save/restore; includes divergence tracking in DEBUG mode

### physics_update
- **Signature:** `static void physics_update(physics_constants *constants, physics_variables *variables, player_data *player, uint32 action_flags)`
- **Purpose:** Core physics simulation; compute new velocity and position without world interaction
- **Inputs:** Physics model constants, player variables, player data (for death state), action flags
- **Outputs/Return:** None; updates variables in-place
- **Side effects:** Modifies velocity, position, elevation, angular velocity, step phase, action state
- **Calls:** (trigonometric lookups via cosine_table/sine_table), `FLOOR()`, `CEILING()`, `PIN()`, `resync_virtual_aim()`, `get_monster_definition_external()`
- **Notes:** Implements movement (forward/back/strafe), turning, looking up/down, jumping/gravity, terminal velocity, auto-recentering; dead players lose control and enter looking_down/up based on velocity direction; very large function (~500 lines) implementing all movement physics

### instantiate_physics_variables
- **Signature:** `static void instantiate_physics_variables(physics_constants *constants, physics_variables *variables, short player_index, bool first_time, bool take_action)`
- **Purpose:** Apply computed physics to game world; collision detection and world updates
- **Inputs:** Physics constants, computed variables, player index, first-time flag, take_action flag
- **Outputs/Return:** None; updates world objects, polygon references, camera location
- **Side effects:** Moves monster/object in world, modifies supporting polygon, updates light sources, triggers polygon change callbacks
- **Calls:** `keep_line_segment_out_of_walls()`, `legal_player_move()`, `translate_map_object()`, `changed_polygon()`, `monster_moved()`, `bump_monster()`, `get_polygon_data()`, `get_media_data()`
- **Notes:** Performs wall/ceiling collision, object collision, polygon boundary crossing; updates player camera location and step bob height; handles media transitions (above/below water)

### process_aim_input
- **Signature:** `uint32 process_aim_input(uint32 action_flags, fixed_yaw_pitch delta)`
- **Purpose:** Convert high-precision aiming input to low-precision network-encoded deltas while tracking fractional error
- **Inputs:** Current action flags, high-precision yaw/pitch delta (from mouse)
- **Outputs/Return:** Updated action_flags with encoded yaw/pitch payloads
- **Side effects:** Updates `vir_aim_delta` to track residual (fractional) aim offset not transmitted over network
- **Calls:** `dont_auto_recenter()`, `std::clamp()`, standard library functions
- **Notes:** Implements classic precision behavior (rounding toward zero instead of nearest); clamps to encoding limits; handles recentering suppression

### get_absolute_pitch_range
- **Signature:** `void get_absolute_pitch_range(_fixed *minimum, _fixed *maximum)`
- **Purpose:** Query pitch limits from current physics model
- **Inputs:** Pointers to store minimum/maximum
- **Outputs/Return:** None; writes to output pointers
- **Side effects:** None
- **Calls:** `get_physics_constants_for_model()`
- **Notes:** Returns negated maximum_elevation as minimum, positive maximum_elevation as maximum

### get_player_forward_velocity_scale
- **Signature:** `_fixed get_player_forward_velocity_scale(short player_index)`
- **Purpose:** Query player's forward speed as fraction of maximum (for animation/display)
- **Inputs:** Player index
- **Outputs/Return:** Fixed-point number in [-FIXED_ONE, FIXED_ONE]
- **Side effects:** None
- **Calls:** `cosine_table[]`, `sine_table[]` lookups
- **Notes:** Used by rendering to select walk/run animation speed; projects velocity onto facing direction

### virtual_aim_delta / resync_virtual_aim
- **Signature:** `fixed_yaw_pitch virtual_aim_delta()` / `void resync_virtual_aim()`
- **Purpose:** Query and reset the virtual aim delta tracking (local player only)
- **Side effects:** `resync_virtual_aim()` zeros `vir_aim_delta`; called when dead, after yaw/pitch controls, or auto-recenter
- **Notes:** Prevents aim twitching when controls override virtual aim

### accelerate_player
- **Signature:** `void accelerate_player(short monster_index, world_distance vertical_velocity, angle direction, world_distance velocity)`
- **Purpose:** Apply external force (knockback, explosions, environmental effects)
- **Inputs:** Monster index, vertical velocity delta, direction, magnitude
- **Outputs/Return:** None; modifies player's external velocity
- **Side effects:** Updates external_velocity components; clamps vertical to terminal velocity
- **Calls:** `get_monster_definition_external()`, trigonometric tables
- **Notes:** For some monsters, sets velocity absolutely instead of adding; special handling for grenade-climbing

## Packing/Unpacking Functions

- **`unpack_physics_constants()` / `pack_physics_constants()`:** Serialize all physics model parameters to/from byte streams for netplay and saves
- **`unpack_m1_physics_constants()`:** Convert Marathon 1 physics format (100 bytes per entry) to current format

## Control Flow Notes

**Initialization:** `initialize_player_physics_variables()` ΓåÆ `instantiate_physics_variables()`

**Per-frame update loop:**
1. Input collected into `action_flags`
2. `process_aim_input()` converts mouse delta to encoded pitch/yaw
3. `update_player_physics_variables()` called:
   - `physics_update()` ΓåÆ new velocity, position, direction, elevation
   - `instantiate_physics_variables()` ΓåÆ collisions, world updates, polygon transitions
4. Rendering reads updated player location and facing

**Physics model selection:** Walking vs. running via `_run_dont_walk` flag in action_flags

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
