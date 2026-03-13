# Source_Files/GameWorld/marathon2.cpp - Enhanced Analysis

## Architectural Role

This file is the **central orchestration hub** of the game engine, coordinating every per-tick world simulation. It acts as the "conductor" that ensures consistent ordering of subsystem updates (lights ΓåÆ media ΓåÆ platforms ΓåÆ players ΓåÆ projectiles ΓåÆ monsters ΓåÆ effects), manages the critical interface between network latency and client-side prediction, and serves as the primary bridge between the input layer (real actions + Lua scripts) and physics simulation. The file also handles the costly level lifecycle (resource loading, monster initialization, Lua setup) and polygon-triggered events that bind map geometry to game logic.

## Key Cross-References

### Incoming (who depends on this file)
- **Shell/Interface**: `initialize_marathon()` called at engine startup; `entering_map()` / `leaving_map()` invoked by level transitions; `update_world()` called repeatedly from main game loop (likely `shell.cpp`)
- **Lua scripting**: Modifiable action queue `GameQueue` exposed via `GetGameQueue()` for Lua trigger modifications (mentioned in comment at line ~100)
- **Network system**: Feeds real player action flags via `GetRealActionQueues()`; Lua overrides via `GetLuaActionQueues()`
- **Player inventory**: Preserves interface state (scrolling position) across prediction rollback to prevent desynchronization

### Outgoing (what this file depends on)
- **All subsystem tickers**: `update_lights()`, `update_medias()`, `update_platforms()`, `update_players()`, `move_projectiles()`, `move_monsters()`, `update_effects()`, `animate_scenery()` (defined elsewhere, called in strict sequence)
- **Lua callbacks**: `L_Call_Idle()` / `L_Call_PostIdle()` / `L_Call_Init()` / `L_Call_Cleanup()` ΓåÉ event-driven extensibility
- **Networking**: `NetProcessMessagesInGame()`, `NetCheckWorldUpdate()`, `NetSync()`, `game_is_over()` ΓåÉ multiplayer synchronization gates
- **Action queue abstraction**: `ModifiableActionQueues` from `ActionQueues.h` ΓåÉ input merging
- **Audio/Music**: `Music::instance()->StopLevelMusic()`, `SoundManager` ΓåÉ resource lifecycle coupling
- **Level state queries**: `check_level_change()`, `live_aliens_on_map()`, `calculate_level_completion_state()` ΓåÉ win/loss conditions

## Design Patterns & Rationale

**Action Queue Multiplexing** (`overlay_queue_with_queue_into_queue`): Merges multiple input sources (real player actions, Lua overrides, prediction rewind) into a single canonical queue feeding the physics engine. This avoids having to thread input sources through every subsystem; instead, all sources converge at one point, and the game loop consumes consistently.

**Partial State Snapshot for Prediction** (`enter_predictive_mode` / `exit_predictive_mode`): Rather than replay the entire world from a checkpoint, only player, monster, and object state is saved (not map geometry, particle effects, or audio). This is a space/speed tradeoff: saves memory vs. full state copy, but requires careful restoration of linked data structures (object lists, parasitic objects). The **interface_flags exception** reveals a discovered quirk: inventory scrolling happens outside the normal input system, so it's preserved across rollback to avoid breaking UX.

**Tick-based Determinism**: All simulation is hardened to 30 FPS ticks, with network/heartbeat gates preventing the engine from running ahead. This ensures reproducibility and simplifies client-side prediction (given the same input, the world advances identically).

## Data Flow Through This File

1. **Input aggregation**: Real action flags + Lua overrides are merged into `GameQueue` by `overlay_queue_with_queue_into_queue()`
2. **Tick gating**: Network (if multiplayer) or heartbeat (if single-player) determines if a tick is allowed; if not, `update_world()` returns early
3. **Main tick sequence**: For each available tick, `update_world_elements_one_tick()` calls subsystems in strict order: lights ΓåÆ media ΓåÆ platforms ΓåÆ players ΓåÆ projectiles ΓåÆ monsters ΓåÆ effects, then checks level-change / game-over conditions
4. **Prediction loop**: If `sPredictionWanted` is true, state is saved, prediction runs ahead consuming future ticks, then rolls back for real updates
5. **Level boundary**: `changed_polygon()` fires when entities cross polygon thresholds, triggering context-sensitive logic (platform entry, monster spawn, light activation)
6. **Output**: Tick count, game time, and damage calculations flow back into the world state for rendering and UI

## Learning Notes

**Idiomatic to this era (early 2000s Marathon/Aleph One):**
- Deterministic tick-based simulation (precedes framerate-decoupling trends)
- Global state for queues and prediction (predates dependency injection)
- Lua as first-class extension (not a retrofit; script callbacks baked into the core loop)
- WAD-based resource model (cf. doom, but with more complex binary layouts)
- Polygon-triggered events (dynamic geometry changes visible to players)

**Modern engines differ by:**
- Frame-independent deltatime physics (this engine locks to 30 FPS)
- ECS / component architecture (this engine uses per-entity struct types)
- Networked replicas via snapshots (this engine predicts client-side, then rolls back)
- Asset streaming (this engine loads entire WAD into memory)

## Potential Issues

1. **Prediction Fragility**: Saving/restoring multiple layers (player ΓåÆ monster ΓåÆ object ΓåÆ parasitic object) creates a tall dependency tower. If any restoration step misses a field, state diverges silently. The `sSavedTickCount` / `sSavedRandomSeed` sanity checks are a band-aid (logged but not fatal).

2. **No Prediction Rollback on Divergence**: If tick count or random seed mismatches are detected (line ~410), the code logs a warning but continues. A networked game experiencing prediction mismatch will experience subtle desynchronization bugs that are hard to reproduce.

3. **Global Action Queue State**: `GameQueue` is a static pointer, making unit testing difficult. A client-side prediction test must mock or reinitialize this global.

4. **Complex Interface Flag Preservation Logic**: The comment at line ~330 admits uncertainty ("I don't know that this is strictly the best place"). If inventory scrolling behavior changes, this preservation logic could break prediction in unexpected ways.

5. **Lua Call Ordering**: `L_Call_Idle()` is called *before* subsystem updates (within `update_world_elements_one_tick`), but `L_Call_PostIdle()` is called *after* (in outer `update_world`). This split is documented in the changelog but creates a potential ordering trap for Lua script authors.

---

*This file is the linchpin of the engine architectureΓÇöany changes to tick ordering, queue management, or prediction logic ripple through multiplayer synchronization and frame consistency. The prediction subsystem in particular is a sophisticated but high-risk feature that would benefit from property-based testing.*
