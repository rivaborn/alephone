# Source_Files/Lua/lua_script.cpp - Enhanced Analysis

## Architectural Role

This file is the **central coordination hub** for game-wide Lua scripting, acting as both a facade to the Lua C API and an event dispatcher that bridges gameplay to user-defined trigger scripts. It manages multiple isolated Lua execution contexts (embedded maps, solo scenarios, networking, achievements) with granular write-access control, enabling modders to react to and modify game state (player/monster actions, platform/light state, item creation) while preventing untrusted scripts from corrupting core world simulation or filesystem access.

## Key Cross-References

### Incoming (who depends on this file)
- **Game world event systems** (`GameWorld/marathon2.cpp`, `monsters.cpp`, `projectiles.cpp`, `items.cpp`, `platforms.cpp`, `devices.cpp`, `lightsource.cpp`): Call trigger methods (`PlayerKilled`, `MonsterKilled`, `MonsterDamaged`, `ItemCreated`, `ProjectileDetonated`, `PlatformActivated`, `LightActivated`, `TagSwitch`, `TerminalEnter`, etc.)
- **Frame loop** (`marathon2.cpp` `idle_game_state()`): Calls `LuaState::Idle()` and `PostIdle()` each tick to fire persistent timers
- **Player damage/death systems**: Query `use_lua_compass`, `can_wield_weapons`, `lua_compass_beacons` globals for HUD state overrides
- **Save/load system** (`game_wad.cpp`): Calls `pack_lua_states()` / `unpack_lua_states()` for persistence across levels
- **Rendering** (`render.cpp`): Calls `UpdateLuaCameras()` / `UseLuaCameras()` for scripted camera animations
- **Input handling** (`player_control` Lua function): Reads `sLuaActionQueues` to apply queued Lua-driven player movement
- **Preferences/plugins**: `LoadSoloLua()`, `LoadStatsLua()`, `LoadAchievementsLua()` called during initialization

### Outgoing (what this file depends on)
- **Lua C API** (wrapped in `extern "C"`): `lua.h`, `lauxlib.h`, `lualib.h` for state creation, stack manipulation, pcall dispatch, library loading
- **Game world state** (`world_view`, `static_world`, `dynamic_world` externs): Camera interpolation reads/writes `world_view->origin`, yaw/pitch; collision queries via `ShootForTargetPoint()`
- **Entity registration modules** (`lua_player.h`, `lua_monsters.h`, `lua_projectiles.h`, `lua_objects.h`, `lua_map.h`, `lua_music.h`): Push object wrappers onto Lua stack for callback arguments
- **Action queue subsystem** (`sLuaActionQueues` global): Enqueues player movement flags from `L_Player_Control`
- **Physics** (`physics_models.h`, `instantiate_physics_variables`): Advanced motion control when ifdef'd `TIENNOU_PLAYER_CONTROL` enabled
- **Game mode/scoring** (`game_scoring_mode`, `game_end_condition` globals): Read by completion state logic
- **Sound** (`SoundManager.h`, `Music.h`): Lua scripts trigger audio via registered functions
- **Achievements** (`achievements.h`): `AchievementsLuaState::set_achievement()` for stat tracking
- **Console** (`Console.h`): `ExecuteCommand()` for REPL-style debug script execution

## Design Patterns & Rationale

**Facade over Lua C API**: `LuaState` class wraps 2500 lines of raw C API calls (stack manipulation, pcall, error handling) into high-level trigger dispatch methods (`GetTrigger`, `CallTrigger`), shielding event callers from Lua details.

**Strategy pattern (write-access control)**: `SoloScriptState` extends `LuaState` with a `SoloLuaWriteAccess` bitmask controlling which subsystems (world/music/fog/ephemera/overlays/sound) are mutable. Prevents malicious solo scripts from modifying incompatible game state.

**Template Method pattern**: Virtual `Initialize()`, `RegisterFunctions()`, `LoadCompatibility()` allow subclasses (`SoloScriptState`, `AchievementsLuaState`) to customize initialization while reusing base setup (persistence table, library loading, security sandboxing).

**Event sourcing**: Game systems call trigger methods instead of directly modifying script state. Trigger methods marshal C++ objects into Lua values, invoke callbacks, catch errors. This decouples game logic from script availabilityΓÇögame continues if a trigger is unregistered or throws.

**Persistence via serialization**: `PassedLuaState` and `SavedLuaState` are `std::map<int, std::string>`, storing opaque serialized Lua table dumps. This allows Lua state to survive level transitions without tight coupling to game save format (binary compatibility via `BStream`).

**Sandbox-first design**: Insecure libraries (`io`, `os`) are conditionally loaded only if `shell_options.insecure_lua` flag set. Functions like `dofile` / `loadfile` are explicitly `nil`-ed in secure mode, forcing scripts to use engine-provided API instead of file I/O.

## Data Flow Through This File

```
Player action (e.g., player dies)
  ΓåÆ GameWorld calls LuaState::PlayerKilled(player_idx, aggressor_idx, ...)
    ΓåÆ GetTrigger("player_killed") searches Lua global Triggers.player_killed
    ΓåÆ Lua_Player::Push(L, player_idx) marshals C++ player object to userdata
    ΓåÆ lua_pcall executes trigger callback
    ΓåÆ Error caught by L_Error(), logged, game continues

Lua script demands player action (L_Player_Control C function)
  ΓåÆ lua_pcall called from script
    ΓåÆ GetLuaActionQueues()->enqueueActionFlags(action)
  ΓåÆ Frame idle loop polls action queue, applies to player physics

Lua state save (level transition)
  ΓåÆ pack_lua_states(): foreach script context, lua_dump() serializes stack
    ΓåÆ SavedLuaState[script_type] = serialized_binary_string
    ΓåÆ Marshalled to game WAD via pack_lua_states(uint8*, size_t)

Lua state restore (level load after save-game)
  ΓåÆ unpack_lua_states(uint8*, size_t) reads binary stream
    ΓåÆ Reconstructs SavedLuaState map
    ΓåÆ Game calls RestoreAll(serialized_string) to lua_load() and execute
```

## Learning Notes

**Error handling philosophy**: Lua runtime errors don't crash the gameΓÇö`lua_pcall` catches them, `L_Error()` logs via `screen_printf()`, and game simulation continues. This is robust for live scripting but differs from modern strict-mode game engines that fail-fast on script errors.

**Implicit single-threaded design**: No mutexes protect `sLuaActionQueues` or trigger invocation, suggesting Lua is always called from main thread. This predates modern async scripting and means long-running Lua scripts can stall frame rate.

**Object marshalling pattern** (e.g., `Lua_Player::Push()`): Each game entity type has a companion Lua binding module that wraps C++ objects as Lua userdata with metatable-driven property access. This avoids exposing raw pointers and enables API versioning.

**Incremental persistence model**: Scripts don't serialize full state to disk; only "passed" state (minimal checkpoint) survives level reload. This was likely a pragmatic choice to keep save files small in 2003, but makes long-term script state fragile across code changes.

**Security layering**: Not a hard sandbox (still vulnerable to Lua interop bypasses), but gating file I/O, OS calls, and require() forces malicious scripts to exploit through game bindings rather than direct system calls.

## Potential Issues

1. **Serialization stability**: `PassedLuaState` stores opaque `lua_dump()` strings. Lua version upgrades or bytecode format changes would silently corrupt saved scriptsΓÇöno versioning header or checksum.

2. **Event dispatch ordering**: Multiple subsystems invoke triggers without explicit orderingΓÇöif script A (invoked by monster death) creates item B, does item B's `ItemCreated` trigger fire same frame or next? Implicit frame boundary assumptions.

3. **Stack safety**: Trigger methods (e.g., `PlayerKilled`) push N arguments then `lua_pcall`ΓÇöif `CallTrigger` fails to pop stack on error path, leak occurs. No RAII wrappers observed in provided excerpt.

4. **Unbounded persistence table**: `L_Persistent_Table_Key()` creates a registry table that scripts can write to indefinitelyΓÇöno quota or garbage collection, risking memory exhaustion in long-play scenarios.

5. **Camera animation interlock**: `UseLuaCameras()` disables weapon HUD when camera activeΓÇöno re-entrancy guard if multiple scripts vie for camera control simultaneously.
