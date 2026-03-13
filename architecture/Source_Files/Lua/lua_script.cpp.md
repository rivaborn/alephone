# Source_Files/Lua/lua_script.cpp

## File Purpose
Implements Lua script loading, execution, and engine integration for Aleph One. Manages multiple script contexts (embedded, solo, netscript, stats, achievements) and dispatches game events to registered Lua callbacks.

## Core Responsibilities
- Load Lua scripts from buffers and files into isolated state instances
- Initialize Lua with standard/sandbox libraries and engine function registrations
- Dispatch game events (player/monster actions, item/projectile interactions, level state) to Lua trigger callbacks
- Manage player control, weapon wielding, compass state, and HUD visibility via Lua
- Serialize/deserialize Lua state persistence across level transitions
- Provide interactive command execution for console/debugging
- Handle resource collection management and camera animation via Lua

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `LuaState` | class | Base Lua state manager with trigger dispatch and lifecycle |
| `SoloScriptState` | class | Solo mode state with granular write-access control (world/music/fog/etc.) |
| `AchievementsLuaState` | class | Sandboxed state for achievement tracking |
| `EmbeddedLuaState`, `NetscriptState`, `StatsLuaState` | typedef | Aliases for LuaState used by different script contexts |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `use_lua_compass` | `bool[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | global | Compass override per player |
| `can_wield_weapons` | `bool[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | global | Weapon permission per player |
| `lua_compass_beacons`, `lua_compass_states` | `world_point2d[]/short[]` | global | Compass display state per player |
| `game_scoring_mode`, `game_end_condition` | `int` | global | Game rules for completion/scoring |
| `mute_lua` | `bool` | global | Suppress Lua message output |
| `states` | `std::map<ScriptType, LuaState*>` | static | Active script instances |
| `PassedLuaState`, `SavedLuaState` | `std::map<int, std::string>` | global | Persistence storage (serialized Lua tables) |
| `sLuaActionQueues` | `ActionQueues*` | static | Player input queue for `player_control` calls |

## Key Functions / Methods

### LuaState::Initialize
- **Purpose**: Set up Lua environment with libraries and engine functions.
- **Inputs**: None (uses `state_` member).
- **Outputs**: Populates Lua registry with standard libraries, persistence table, and registered C functions.
- **Side effects**: Loads base/table/string/math/debug libraries; conditionally disables `dofile`/`loadfile` if secure mode; registers compatibility triggers.
- **Calls**: `luaL_requiref()`, `RegisterFunctions()`, `LoadCompatibility()`.
- **Notes**: `SoloScriptState` additionally loads `io` library; `AchievementsLuaState` registers `set_achievement()`.

### LuaState::Run
- **Purpose**: Execute all loaded script chunks in order.
- **Inputs**: None; reads `num_scripts_`, `state_`.
- **Outputs**: Sets `running_` to true if successful; returns success status.
- **Side effects**: Calls all loaded Lua code chunks; on error, calls `L_Error()` with error message.
- **Calls**: `lua_pcall()`, `L_Error()`.
- **Notes**: Reverses function order before execution; stops at first runtime error.

### LuaState::GetTrigger / CallTrigger
- **Purpose**: Dispatch a named trigger from the Lua `Triggers` table.
- **Inputs**: `trigger` (string name); `numArgs` for CallTrigger.
- **Outputs**: Boolean (trigger found); stack manipulated for pcall.
- **Side effects**: Calls Lua function via `lua_pcall()`; on error invokes `L_Error()`.
- **Calls**: `lua_getglobal()`, `lua_gettable()`, `lua_pcall()`.
- **Notes**: Returns false if not running, trigger not found, or not a function.

### Event Trigger Methods (e.g., PlayerKilled, MonsterDamaged)
- **Signature**: `void LuaState::PlayerKilled(short player_index, short aggressor_player_index, short action, short projectile_index)`
- **Purpose**: Invoke registered Lua `player_killed` callback with game event details.
- **Inputs**: Entity indices, action type, optional projectile index.
- **Outputs**: None; triggers Lua callback with pushed arguments.
- **Side effects**: Pushes objects onto Lua stack, calls Lua function, pops stack.
- **Calls**: `GetTrigger()`, `Lua_Player::Push()`, `Lua_MonsterAction::Push()`, `CallTrigger()`.
- **Notes**: Pushes `nil` for optional indices when -1; pattern repeats for ~20 event types.

### L_Player_Control
- **Signature**: `int L_Player_Control(lua_State *L)`
- **Purpose**: Queue player actions (movement, turning, looking) from Lua.
- **Inputs**: `(player_index, move_type, value, ...)`; varies by move_type.
- **Outputs**: Returns 0.
- **Side effects**: Allocates `sLuaActionQueues` if needed; enqueues action flags to queue.
- **Calls**: `GetLuaActionQueues()->enqueueActionFlags()`, physics functions (if `TIENNOU_PLAYER_CONTROL`).
- **Notes**: Extensive ifdef'ed code; non-ifdef path supports basic movement flags; ifdef'd path allows complex motion planning (partially commented).

### pack_lua_states / unpack_lua_states
- **Purpose**: Serialize/deserialize Lua state persistence (used for save/load).
- **Inputs**: `pack_lua_states(uint8* data, size_t length)`; mirror signature for unpack.
- **Outputs**: Serialized state as binary (pack); reconstructed maps (unpack).
- **Side effects**: Writes to/reads from binary stream; clears SavedLuaState on pack.
- **Calls**: `BOStreamBE`/`BIStreamBE` stream operators.
- **Notes**: Stores ScriptType key (int16), state size (uint32), raw serialized string data.

### UpdateLuaCameras / UseLuaCameras
- **Purpose**: Animate scripted camera paths and apply to world view.
- **Inputs**: `UpdateLuaCameras()` advances time; `UseLuaCameras()` reads current state.
- **Outputs**: Boolean (cameras in use); modifies `world_view->origin`, yaw/pitch if active.
- **Side effects**: Updates timing counters, interpolates camera positions/angles, disables weapon HUD.
- **Calls**: `FindLinearValue()`, `ShootForTargetPoint()`, `normalize_angle()`.
- **Notes**: Only applies to active local player's cameras; per-frame check for point/angle advancement.

### LoadSoloLua / LoadStatsLua / LoadAchievementsLua
- **Purpose**: Load optional Lua scripts from preferences or plugins.
- **Inputs**: File names and directories from config.
- **Outputs**: Loaded and initialized LuaState in `states` map.
- **Side effects**: Opens files, reads into buffers, constructs state, sets search path.
- **Calls**: `_LoadLuaScript()`, `FileSpecifier::Open()`, `SetSearchPath()`.
- **Notes**: `LoadAchievementsLua()` disables achievements if other scripts present; `LoadStatsLua()` queries plugin system.

## Control Flow Notes
- **Initialization**: `initialize_game_state()` (implicit) ΓåÆ scripts loaded at level start.
- **Per-frame**: `idle_game_state()` calls `LuaState::Idle()` ΓåÆ triggers run each tick.
- **Event-driven**: Game systems call trigger methods (e.g., monster death ΓåÆ `MonsterKilled()`).
- **Shutdown**: `CloseLuaScript()` saves state, clears `states` map, restores engine settings.
- **Persistence**: `save_lua_states()` serializes at save-game; `unpack_lua_states()` deserializes on load.

## External Dependencies
- **Lua C API**: `lua.h`, `lauxlib.h`, `lualib.h` for state management and stack operations.
- **Lua modules** (this file): `lua_music.h`, `lua_ephemera.h`, `lua_map.h`, `lua_monsters.h`, `lua_objects.h`, `lua_player.h`, `lua_projectiles.h` for object registration.
- **Game engine**: `world.h`, `player.h`, `monsters.h`, `items.h`, `weapons.h`, `platforms.h`, `render.h`, `physics_models.h`.
- **Extern symbols**: `dynamic_world`, `static_world`, `world_view` (global engine state); `local_player_index` (active player); `local_random()` (RNG); `ShootForTargetPoint()`, `get_physics_constants_for_model()`, `instantiate_physics_variables()` (physics).
