# Source_Files/Network/network_games.cpp - Enhanced Analysis

## Architectural Role

This file implements the **multiplayer game mode ruleset engine**, sitting at the critical junction between world simulation (GameWorld), UI rendering, and scripting. It's responsible for translating low-level player/team statistics into game-mode-specific rankings and objectives. Rather than enforcing rules during gameplay, it post-processes state: players accumulate damage/possession naturally; this file *interprets* those facts through the lens of each game mode. This separation allows game rules to be orthogonal to physics simulation.

## Key Cross-References

### Incoming (who depends on this file)

- **marathon2.cpp** (`update_world` loop): Calls `update_net_game()` once per 30 FPS tick to increment mode-specific counters; calls `game_is_over()` to check end conditions
- **screen.cpp** / HUD rendering: Calls `get_player_net_ranking()`, `calculate_player_rankings()`, `calculate_ranking_text()` to format live scores
- **game_window.h**: Calls `mark_player_network_stats_as_dirty()` to signal UI refresh when stats change
- **network_messages.cpp**: Likely reads `team_netgame_parameters` for syncing team state across network peers
- **weapons.c / items.cpp**: Calls `player_killed_player()` to handle Tag mode "it" transfers; calls `update_net_game()` during flag/ball interaction checks

### Outgoing (what this file depends on)

- **GameWorld**: Reads `dynamic_world->player_count`, `dynamic_world->game_information`, `dynamic_world->game_player_index` (the "it" player in Tag); queries `get_player_data()`, `get_polygon_data()` for state
- **map.h**: Queries polygon types (`_polygon_is_hill`, `_polygon_is_base`, supports goal detection); reads polygon center and player supporting polygon
- **player.h**: Reads `player_data.damage_taken[index]`, `player_data.monster_damage_given/taken`, `player_data.netgame_parameters[]`, `player_data.team`
- **items.h**: Calls `find_player_ball_color()`, `player_has_ball()`; calls external `destroy_players_ball()`
- **lua_script.h**: Reads `use_lua_compass[]`, `lua_compass_beacons[]`, `lua_compass_states[]`; calls `GetLuaScoringMode()`, `GetLuaGameEndCondition()` for custom game logic
- **SoundManager.h**: Calls `play_object_sound()` for Tag mode "you are it" sound
- **game_window.h**: Calls `mark_player_network_stats_as_dirty()`

## Design Patterns & Rationale

### Per-Game-Mode Dispatch Switch

Every major function (`get_player_net_ranking`, `update_net_game`, `get_network_compass_state`) contains a `switch(GET_GAME_TYPE())` statement. This is **explicit game-mode dispatch** ΓÇö clean separation of rules but high code duplication. Modern engines would use polymorphism or data-driven configuration; this approach reflects late-1990s design where each game mode was a distinct feature.

**Rationale**: Avoids vtable overhead; makes mode logic concentrated and auditable; reflects original Marathon's monolithic design philosophy.

### Dual Player/Team Ranking Functions

`get_player_net_ranking()` and `get_team_net_ranking()` are nearly identical structures with parallel logic. This is a **code smell** suggesting a missing abstraction (template or strategy pattern). However, the team version aggregates `team_damage_given/taken` globals while the player version sums across damage records ΓÇö the data structures differ enough that DRY refactoring would obscure rather than clarify.

### Lua Compass Override

`get_network_compass_state()` first checks `use_lua_compass[player_index]`. If set, it *completely bypasses* the game-mode-specific beacon logic. This shows **scripting-first extensibility**: Lua scripts can entirely replace compass behavior for custom game modes without modifying engine code. This is a rare pattern for 1990s engines; suggests Lua integration was high-priority.

### Per-Tick Accumulation + Query Model

Game state is accumulated during `update_net_game()` (increments `netgame_parameters[_time_spent_it]`, etc.), then queried during ranking calculation and HUD rendering. This **defers interpretation** ΓÇö the engine doesn't know or care what "time spent it" means until it's formatted. Clean separation: world ticks blindly, UI reads results.

## Data Flow Through This File

```
Input:
  - Player damage records (player.damage_taken[index].kills)
  - Ball/flag possession (find_player_ball_color)
  - Polygon types (polygon->type == _polygon_is_hill)
  - Player location (player->location)
  - Lua overrides (lua_compass_beacons, GetLuaScoringMode)

Transform:
  1. update_net_game(): Aggregate possession time, goal detection, objectives
  2. get_player_net_ranking(): Convert mode-specific stat to ranking value
  3. get_network_compass_state(): Calculate bearing to objective

Output:
  - Ranking values (displayed on scoreboard)
  - Compass quadrants (drawn on HUD)
  - Game-over trigger (stops main loop)
  - Team parameter updates (synced to peers)
```

Example flow for King of the Hill:
1. Polygon marked `_polygon_is_hill` during map load
2. `initialize_net_game()` finds hill center ΓåÆ `dynamic_world->game_beacon`
3. Each tick: `update_net_game()` checks if player in hill ΓåÆ increments `netgame_parameters[_king_of_hill_time]`
4. UI calls `get_network_compass_state()` ΓåÆ calculates angle to beacon ΓåÆ sets compass quadrants
5. At end: `get_player_net_ranking()` returns accumulated time for scoreboard

## Learning Notes

**Idiomatic to this era:**
- **No event systems**: Rules are checked manually during update loop, not via callbacks. Modern engines post death/kill events to a queue.
- **Explicit polygon type checks**: Relies on map designer tagging polygons (`_polygon_is_base`, `_polygon_is_hill`). Modern systems would use zones or trigger volumes with metadata.
- **Global state accumulation**: `team_netgame_parameters` is a global array, not encapsulated per game instance. Assumes one active game at a time.
- **Compass beacon pattern**: The idea of an on-screen compass pointing to an objective is Marathon-specific; later games moved to icons/radar/nav meshes.

**Interesting modern contrast:**
- Lua compass override shows *early* metaprogramming integration (pre-2000s engines rarely exposed compass logic to scripts)
- The `GetLuaScoringMode()` pattern ΓÇö delegating entire scoring to Lua for custom games ΓÇö is forward-thinking but now standard in all engines

## Potential Issues

1. **Polygon type assumptions**: If a KOTH map has zero `_polygon_is_hill` polygons, the beacon initializes to (0, 0), which may coincide with valid map locations. Silent failure.

2. **Compass bearing cast**: Line `beacon= (world_point2d *) &get_player_data(dynamic_world->game_player_index)->location` assumes the `location` field is at the struct start. C++ doesn't guarantee this; if the struct layout changes, this becomes undefined behavior.

3. **Defense mode logic complexity**: The nested loops in `get_player_net_ranking()` and `get_team_net_ranking()` for Defense mode recalculate the max offender time every call. Inefficient; should be cached or recalculated only on state changes. Comment says "Bogus for now" ΓÇö may be incomplete.

4. **No bounds checking on netgame_parameters indexing**: Arrays like `player->netgame_parameters[_offender_time_in_base]` are indexed by enum constants. If a mode-specific enum is used on wrong game type, silent integer access to undefined field.

5. **Lua compass state arrays not bounds-checked**: `lua_compass_states[player_index]` assumes `player_index < MAXIMUM_NUMBER_OF_NETWORK_PLAYERS`. If a player joins during iteration and index shifts, potential out-of-bounds read.

6. **game_is_over() side effects**: First call disables the kill limit flag and sets a 2-second grace period. Hidden state mutation makes testing and debugging harder; should be explicit or logged.
