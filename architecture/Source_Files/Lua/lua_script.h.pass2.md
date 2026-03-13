# Source_Files/Lua/lua_script.h - Enhanced Analysis

## Architectural Role

This file is the **public interface contract** between Aleph One's core engine subsystems and the embedded Lua scripting runtime. It serves as a bidirectional bridge: GameWorld subsystems dispatch events through `L_Call_*` callbacks, while Lua scripts inspect/modify game state and inject input via ActionQueues. The file also handles state serialization (critical for save/load and network replication) and provides specialized systems like scripted camera paths for cutscenes and achievement trackingΓÇömaking Lua a first-class gameplay extension mechanism rather than a debug tool.

## Key Cross-References

### Incoming (who depends on this file)

**GameWorld entity systems** call event callbacks:
- `marathon2.cpp` (main loop): calls `L_Call_Idle`, `L_Call_PostIdle`, `L_Call_Calculate_Completion_State` each frame
- `monsters.cpp`: calls `L_Call_Monster_Killed`, `L_Call_Monster_Damaged`, `L_Call_Monster_Kamikazed`
- `projectiles.cpp`: calls `L_Call_Projectile_Created`, `L_Call_Projectile_Detonated`
- `items.cpp`: calls `L_Call_Got_Item`, `L_Call_Item_Created`
- `devices.cpp` (switches/terminals): calls `L_Call_Tag_Switch`, `L_Call_Light_Switch`, `L_Call_Platform_Switch`, `L_Call_Terminal_Enter/Exit`
- `lightsource.cpp`, `platforms.cpp`: calls `L_Call_Light_Activated`, `L_Call_Platform_Activated`
- `player.cpp`: calls `L_Call_Player_Killed`, `L_Call_Player_Revived`, `L_Call_Player_Damaged`, `L_Call_Got_Item`

**HUD/Screen rendering** uses camera and stats:
- `RenderOther/computer_interface.cpp`: queries completion state
- Achievement/stats display systems: calls `LoadAchievementsLua`, `LoadStatsLua`, `CollectLuaStats`

**Network/persistence layer**:
- `game_wad.cpp`: calls `unpack_lua_states`, `pack_lua_states`, `save_lua_states` during save/load/network sync

### Outgoing (what this file depends on)

- **ActionQueues** (included): Lua injects player commands; `GetLuaActionQueues()` returns the queue handler
- **CSeries** types: Platform abstraction, basic types
- **World** types: `world_point3d`, polygon indices, entity coordinates
- **Shape descriptors**: Texture palette and rendering assets
- **Files subsystem** (implicit): LoadLuaScript buffers come from WAD/archive loading
- **Rendering subsystem** (implicit): Camera paths affect view calculations; texture palette affects scene rendering
- **Sound subsystem** (implied via `LuaMutabilityInterface::music_mutable`): Lua can toggle/query sound mutability

## Design Patterns & Rationale

**Observer/Event Dispatch Pattern**: The 20+ `L_Call_*` functions implement a publish-subscribe model where GameWorld subsystems are loosely coupled from Lua script logic. GameWorld never includes Lua headers directly; it just calls these exported functions. This allows Lua functionality to be toggled, replaced, or disabled without recompiling core systems.

**Mutability Gates**: The `LuaMutabilityInterface` abstract class is a permission systemΓÇöLua scripts can query which game systems they're allowed to modify at any given moment (e.g., no world mutation during intro/loading). This enforces immutability guarantees without runtime cost checks on every Lua call.

**State Machines for Script Lifecycle**: Scripts are `Load`ed, `Run`, `Reset`, and eventually `Close`d, mirroring Unix file descriptor patterns. This design avoids global Lua VM state issues by treating scripts as ephemeral resources.

**Dual Path System for Cameras**: `lua_path` separates position waypoints from orientation (yaw/pitch) with independent playback indices. This decouples position and look-around for smoother cutscenes and allows dynamic camera rigs.

## Data Flow Through This File

```
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  FILE INGESTION (load phase)                             Γöé
Γö£ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
Γöé  LoadLuaScript(buffer, len, type) ΓöÇΓöÇΓåÆ Lua VM init       Γöé
Γöé  RunLuaScript() ΓöÇΓöÇΓåÆ Lua sandbox execution               Γöé
Γöé  LoadSoloLua/LoadReplayNetLua/LoadAchievementsLua       Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                           Γåô
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  RUNTIME LOOP (frame-by-frame)                           Γöé
Γö£ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
Γöé  L_Call_Idle() ΓöÇΓöÇΓåÆ per-frame Lua tick                   Γöé
Γöé  [event callbacks if entities change/die/pickup]        Γöé
Γöé  L_Call_PostIdle() ΓöÇΓöÇΓåÆ post-logic Lua updates           Γöé
Γöé  L_Calculate_Completion_State() ΓöÇΓöÇΓåÆ query win/lose      Γöé
Γöé  GetLuaActionQueues() ΓöÇΓöÇΓåÆ Lua-injected player actions   Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
                           Γåô
ΓöîΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé  STATE PERSISTENCE (save/load/network)                   Γöé
Γö£ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
Γöé  save_lua_states() ΓöÇΓöÇΓåÆ size hint                         Γöé
Γöé  pack_lua_states(buffer, len) ΓöÇΓöÇΓåÆ serialize to WAD      Γöé
Γöé  unpack_lua_states(buffer, len) ΓöÇΓöÇΓåÆ restore from save   Γöé
ΓööΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÿ
```

**Invalidation Flow** (prevents use-after-free):
- Entity destroyed (monster killed, projectile detonated) ΓåÆ `L_Invalidate_*` called ΓåÆ Lua clears its reference ΓåÆ prevents crashes if Lua tries to access freed object

## Learning Notes

1. **Lua as a Core Gameplay System** (not a scripting afterthought):
   - Aleph One treats Lua as a peer to C++ systems, not a debug layer. The extensive event callback API shows Lua is designed to fully control or override game rules (custom scoring modes, win conditions, input injection).

2. **Observer Pattern Before Modern C++**:
   - Pre-C++11 codebase; no virtual callbacks or `std::function` for event dispatch. The explicit callback list (`L_Call_Init`, `L_Call_Idle`, etc.) is more maintainable than a callback registry for a ~2003-era engine.

3. **Cutscene Camera Design**:
   - The `lua_camera` / `lua_path` structures separate path waypoints from angle rotations. This is cleaner than a unified 6D spline and mirrors theatrical camera rigs (move camera position, pan/tilt separately).

4. **Serialization as a First-Class Citizen**:
   - Unlike many engines that hide Lua state serialization, Aleph One exposes `pack/unpack_lua_states` at the file interface level. This is essential for save games, replays, and network replication to deterministically resume Lua execution.

5. **Mutability as a Temporal Permission System**:
   - The `LuaMutabilityInterface` enforces that Lua can't corrupt immutable game states (e.g., can't rewrite the map while it's being rendered). This is a design pattern worth notingΓÇönot just a simple boolean flag, but a permission interface queried at runtime.

## Potential Issues

1. **Silent Event Dispatch Failures**: If a GameWorld subsystem forgets to call a required `L_Call_*` callback (e.g., missing `L_Call_Monster_Killed` in one code path), Lua scripts silently miss events with no compile-time or runtime error. The extensive callback list is error-prone.

2. **Pointer Ownership Ambiguity**: `GetLuaActionQueues()` returns a raw `ActionQueues*` with no documentation of lifetime or ownership. If the queue is freed while Lua holds a reference, a dangling pointer results.

3. **State Persistence Format Brittleness**: `pack_lua_states` / `unpack_lua_states` imply binary serialization of Lua VM state. If Lua version changes or table layout shifts, old save games silently corrupt or crash during unpack. No version tag or CRC is mentioned.

4. **Circular Dependency Risk**: GameWorld calls into Lua (event callbacks), and Lua queries GameWorld (via ActionQueues + getters). If Lua modifies game state during an event callback while GameWorld is mid-update, invariants can break (e.g., platform geometry changed mid-physics tick).
