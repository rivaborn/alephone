# Source_Files/Lua/lua_player.cpp

## File Purpose
Implements Lua bindings for all player-related game objects in Aleph One, exposing player state (position, velocity, orientation, inventory, weapons, energy) to Lua scripts. Provides a template-based bridge between C++ game code and Lua scripting through reader/writer function registration.

## Core Responsibilities
- Register player, game state, camera, overlay, and compass objects with Lua VM
- Provide getter/setter methods for player properties (position, velocity, orientation, energy, oxygen, items, weapons)
- Manage action flag (input) queues accessible to Lua during `idle()` callbacks
- Implement camera path animation system with waypoint and angle keyframe management
- Expose player HUD overlays (text, icons, colors) to script control
- Provide inventory and weapon management with accounting (item creation/destruction tracking)
- Support game state queries and mutations (difficulty, game type, scoring mode, time remaining)
- Serialize/deserialize game state to/from binary format
- Maintain backwards compatibility with legacy Lua scripts via compatibility wrapper functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `lua_camera` | struct | Represents an animatable camera with path points, angles, and playback state |
| `lua_camera::path_data` | struct | Contains waypoints and angle keyframes with current playback index and elapsed time |
| `lua_texture` | struct | Texture palette slot containing shape descriptor and type |
| `PlayerSubtable<name>` | template class | Base class for per-player objects (overlays, texture palettes) storing player index |
| `Lua_Action_Flags` | L_Class typedef | Binding for reading/modifying queued input flags (weapon cycle, trigger states, etc.) |
| `Lua_Player` | L_Class typedef | Binding for player entity with position, velocity, orientation, energy, inventory |
| `Lua_Game` | L_Class typedef | Singleton binding for global game state (difficulty, type, ticks, time remaining) |
| `Lua_Overlays` | L_Class typedef | Container providing indexed access to player HUD overlay elements |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `lua_cameras` | `vector<lua_camera>` | extern | Active camera instances; indexed by Lua scripts |
| `lua_texture_palette` | `vector<lua_texture>` | static | Texture palette slots for current session |
| `lua_texture_palette_selected` | `int` | static | Index of currently selected palette slot (ΓêÆ1 if none) |
| `use_lua_compass[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | `bool[]` | extern | Per-player flag enabling Lua-controlled compass override |
| `lua_compass_beacons[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | `world_point2d[]` | extern | Beacon positions for Lua compass display |
| `lua_compass_states[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | `short[]` | extern | Per-player compass direction flags (NE/NW/SE/SW/beacon) |
| `AngleConvert` | `const float` | file-static | Conversion factor: 360 / FULL_CIRCLE for angle normalization |

## Key Functions / Methods

### Lua_Player_register
- Signature: `int Lua_Player_register(lua_State *L, const LuaMutabilityInterface& m)`
- Purpose: Main registration function; sets up all player-related Lua bindings based on mutability interface permissions
- Inputs: Lua state, mutability settings (world_mutable, sound_mutable, overlays_mutable)
- Outputs/Return: 0 (void return via Lua stack)
- Side effects: Registers ~30 classes/containers in Lua VM; creates global `Game` singleton; loads compatibility script
- Calls: `Register()`, `RegisterAdditional()`, `Push()` methods on template classes; `Lua_Player_load_compatibility()`
- Notes: Called once at startup; mutability interface controls which getters/setters are exposed (e.g., world-mutable-only setters hidden in read-only contexts)

### Lua_Player_Items_Set
- Signature: `static int Lua_Player_Items_Set(lua_State *L)`
- Purpose: Set player item count with accounting and special handling for weapons/balls
- Inputs: L (Lua stack), arg 1 = player index, arg 2 = item type, arg 3 = new count
- Outputs/Return: 0 (no return value)
- Side effects: Modifies player inventory; marks inventory dirty; calls `try_and_add_player_item()` or `destroy_players_ball()`; invokes accounting if enabled
- Calls: `get_player_data()`, `get_item_definition_external()`, `find_player_ball_color()`, `destroy_players_ball()`, `select_next_best_weapon()`, `try_and_add_player_item()`, `mark_player_inventory_as_dirty()`, `object_was_just_destroyed()`, `object_was_just_added()`
- Notes: Clamps new_item_count ΓëÑ 0; converts 0 to NONE sentinel; handles balls separately (destroy rather than decrement); triggers weapon selection if equipped weapon removed

### Lua_Camera_Path_Points_New
- Signature: `int Lua_Camera_Path_Points_New(lua_State *L)`
- Purpose: Append waypoint to camera animation path
- Inputs: L, arg 1 = camera index, args 2ΓÇô4 = x/y/z position (world units), arg 5 = polygon (int or Polygon object), arg 6 = time delta (ms)
- Outputs/Return: 0 (no return)
- Side effects: Resizes `lua_cameras[camera_index].path.path_points` vector; appends new waypoint with angle interpolation
- Calls: `Lua_Camera::Index()`, `Lua_Polygon::Valid()`, `Lua_Polygon::Is()`, `Lua_Polygon::Index()`
- Notes: Scales position by WORLD_ONE; validates polygon; inverts AngleConvert (divide, not multiply) for scale conversion

### Lua_Game_Serialize / Lua_Game_Deserialize
- Signature: `int Lua_Game_Serialize(lua_State *L)` / `int Lua_Game_Deserialize(lua_State *L)`
- Purpose: Serialize entire game state to/from binary string for save/load
- Inputs: L; deserialize takes binary string from arg 1
- Outputs/Return: 1 (pushes serialized string or boolean success)
- Side effects: Calls `lua_save()` / `lua_restore()` which traverse and serialize/restore Lua state
- Calls: `lua_save()`, `lua_restore()` (defined elsewhere)
- Notes: Uses Boost iostreams `stringbuf` and `stream_buffer` for memory I/O; restores all player state, inventory, and script variables

### Lua_Player_Position / Lua_Player_Teleport
- Signature: `static int Lua_Player_Position(lua_State *L)` / `static int Lua_Player_Teleport(lua_State *L)`
- Purpose: Set player position in world; teleport applies fade effect
- Inputs: L, arg 1 = player index, args 2ΓÇô4 = x/y/z, arg 5 (optional) = polygon index
- Outputs/Return: 0 (void)
- Side effects: Modifies player position; marks physics variables dirty; teleport includes fade and sound
- Calls: `get_player_data()`, `instantiate_physics_variables()`, `play_teleport_sound()`, `start_fade()`
- Notes: Scales coords by WORLD_ONE; clamps z to 0..15000; resync_virtual_aim() called for local player only

### Lua_Player_Damage
- Signature: `static int Lua_Player_Damage(lua_State *L)`
- Purpose: Inflict damage to player
- Inputs: L, arg 1 = player index, arg 2 = amount, arg 3 (optional) = damage type
- Outputs/Return: 0
- Side effects: Applies damage via `damage_player()`; may trigger death or stun effects
- Calls: `get_player_data()`, `damage_player()`, `Lua_DamageType::ToIndex()`
- Notes: Default damage type is 0; called from Lua during game events

---

## Control Flow Notes
This file implements the **Lua binding layer** and does not have traditional frame/update/render flow. Instead:
- **Initialization**: `Lua_Player_register()` runs once at game startup, registering all object types and methods with the Lua VM.
- **Script execution**: When Lua scripts call `Players[i].property` or `Game.method()`, the registered getter/setter functions are invoked synchronously by the Lua VM.
- **Action flags**: During `idle()` callbacks (game loop idle phase), Lua scripts read/modify action flags via `Lua_Action_Flags_Get_t<>` and `Lua_Action_Flags_Set_t<>`, which directly query/modify the action queue.
- **Camera updates**: Camera path animations are updated elsewhere (not in this file); this file only allows scripts to define and activate paths.
- **Serialization**: Triggered on explicit `Game.save()` / `Game.deserialize()` calls; state is persisted to disk or string.

## External Dependencies
- **ActionQueues.h**: `GetGameQueue()`, `countActionFlags()`, `peekActionFlags()`, `modifyActionFlags()` for input queue management
- **game state**: `dynamic_world`, `static_world`, `current_player_index`, `local_player_index` globals
- **player.h** (implied): `player_data`, `get_player_data()`, `damage_player()`, `mark_player_inventory_as_dirty()`
- **item_definitions.h**: `get_item_definition_external()`, `try_and_add_player_item()`, `item_definition` struct
- **monsters.h / projectiles.h**: Monster/projectile entity queries
- **map.h**: Polygon validation
- **lua_templates.h**: Template base classes (`L_Class<>`, `L_Container<>`, `L_Enum<>`) for Lua binding abstraction
- **screen.h**: `screen_printf()` for console output
- **Crosshairs.h**: `Crosshairs_IsActive()`, `Crosshairs_SetActive()`
- **game_window.h**: `mark_shield_display_as_dirty()`, `draw_panels()`
- **SoundManager.h**: `play_teleport_sound()`
- **fades.h**: `start_fade()`
- **ViewControl.h**: `resync_virtual_aim()`, `SetTunnelVision()`
- **network.h** / **network_games.h**: Networking state, team data
- **lua_map.h, lua_monsters.h, lua_objects.h**: Map/monster/object Lua bindings
- **Boost iostreams**: Memory-based I/O for serialization
