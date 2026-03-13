# Source_Files/Lua/lua_script.h

## File Purpose
Header for Lua scripting subsystem in Aleph One (Marathon game engine). Provides event callbacks for game world interactions, script lifecycle management, state persistence, and camera/cutscene control via Lua scripts.

## Core Responsibilities
- Script lifecycle (load, execute, cleanup, reset)
- Event dispatch for game interactions (damage, kills, switches, terminals, projectiles, items)
- State invalidation tracking (effects, monsters, projectiles, objects, ephemera)
- Lua state serialization/deserialization for save/load
- Camera path management for scripted cutscenes
- Game rule control (scoring modes, end conditions)
- Mutability enforcement (restricting Lua access to game systems)
- Achievement and stats collection
- Action queue management for input handling

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ScriptType` | enum | Categorizes script types (embedded, netscript, solo, stats, achievements) |
| `timed_point` | struct | Position + duration for camera waypoints |
| `timed_angle` | struct | Yaw/pitch rotation + duration for camera orientation |
| `lua_path` | struct | Complete camera path with waypoint/angle sequences and current playback indices |
| `lua_camera` | struct | Active camera state tracking position along a path |
| `LuaMutabilityInterface` | abstract class | Interface checking which game systems (world, sound, music, fog, overlays, ephemera) Lua can modify |

## Global / File-Static State
None.

## Key Functions / Methods

### L_Call_Init
- Signature: `void L_Call_Init(bool fRestoringSaved)`
- Purpose: Initialize Lua subsystem and load initial scripts
- Inputs: `fRestoringSaved` ΓÇô whether restoring from a saved game
- Side effects: Loads and executes initialization scripts
- Notes: Called once per level/game start

### L_Call_Idle / L_Call_PostIdle
- Signature: `void L_Call_Idle()` / `void L_Call_PostIdle()`
- Purpose: Per-frame Lua script updates
- Outputs/Return: None
- Notes: Idle called before frame logic; PostIdle called after

### Event Callbacks (L_Call_*)
Entry points for scripted reactions to game events:
- **Switches**: `L_Call_Tag_Switch`, `L_Call_Light_Switch`, `L_Call_Platform_Switch`, `L_Call_Projectile_Switch`
- **Refueling**: `L_Call_Start_Refuel`, `L_Call_End_Refuel`
- **Terminals**: `L_Call_Terminal_Enter`, `L_Call_Terminal_Exit`, `L_Call_Pattern_Buffer`
- **Damage**: `L_Call_Player_Damaged`, `L_Call_Monster_Damaged`
- **Deaths**: `L_Call_Player_Killed`, `L_Call_Monster_Killed`, `L_Call_Player_Revived`
- **Projectiles**: `L_Call_Projectile_Created`, `L_Call_Projectile_Detonated`
- **Items**: `L_Call_Got_Item`, `L_Call_Item_Created`
- **World**: `L_Call_Light_Activated`, `L_Call_Platform_Activated`, `L_Call_Monster_Kamikazed`

### LoadLuaScript / RunLuaScript
- Signature: `void LoadLuaScript(const char *buffer, size_t len, ScriptType type)` / `bool RunLuaScript()`
- Purpose: Load script from memory buffer and execute it
- Inputs: Script buffer, length, script category
- Outputs/Return: Success bool for RunLuaScript
- Side effects: Initializes Lua VM state
- Notes: Type determines script behavior/permissions

### Invalidation Functions (L_Invalidate_*)
- Signature: `void L_Invalidate_{Effect,Monster,Projectile,Object,Ephemera}(short index)`
- Purpose: Notify Lua that a game object has been destroyed
- Inputs: Entity index
- Side effects: Lua clears references to that object
- Notes: Prevents use-after-free bugs in Lua scripts

### State Persistence
- Signature: `void unpack_lua_states(uint8* data, size_t length)` / `void pack_lua_states(uint8* data, size_t length)` / `size_t save_lua_states()`
- Purpose: Serialize/deserialize Lua VM state for save/load
- Inputs/Outputs: Data buffer and length
- Side effects: Modifies Lua VM state or writes to buffer
- Notes: `save_lua_states()` returns size needed

### L_Calculate_Completion_State
- Signature: `bool L_Calculate_Completion_State(short& completion_state)`
- Purpose: Query Lua for custom level completion conditions
- Inputs/Outputs: Reference to completion state (filled by Lua)
- Return: Success bool
- Notes: Allows Lua to override/extend win/loss conditions

### GetLuaActionQueues
- Signature: `ActionQueues* GetLuaActionQueues()`
- Purpose: Retrieve action queue handler for Lua-controlled input
- Outputs/Return: Pointer to action queue manager
- Notes: Allows Lua to inject player commands

## Control Flow Notes
- **Init phase**: `L_Call_Init` during level setup
- **Frame loop**: `L_Call_Idle` ΓåÆ game logic ΓåÆ `L_Call_PostIdle` each frame
- **Event dispatch**: Game subsystems call relevant `L_Call_*` callbacks on events
- **Cleanup**: `L_Call_Cleanup` during shutdown or level transition
- Lua execution is gated by `LuaMutabilityInterface` (checks what systems Lua can access at current game state)

## External Dependencies
- `cseries.h` ΓÇô platform/utility definitions
- `world.h` ΓÇô world coordinates (`world_point3d`), angles
- `ActionQueues.h` ΓÇô input queue management (`ActionQueues` class)
- `shape_descriptors.h` ΓÇô sprite/image descriptors
- Standard C++: `<map>`, `<string>`
