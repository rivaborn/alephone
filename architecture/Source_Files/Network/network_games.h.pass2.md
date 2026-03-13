# Source_Files/Network/network_games.h - Enhanced Analysis

## Architectural Role

This module serves as the **game-mode abstraction layer** translating raw player damage/death events into game-mode-specific ranking semantics and UI state. It sits between GameWorld (which tracks kills at the physics/damage level) and HUD/UI renderers that display scores/rankings. The module handles the fundamental tension in networked multiplayer: each game type (deathmatch, CTF, king-of-the-hill, etc.) interprets the same underlying events differently, so `network_games.h` abstracts those interpretations behind a unified ranking API.

## Key Cross-References

### Incoming (who depends on this file)
- **HUD/UI rendering systems**: Call `calculate_player_rankings()`, `get_player_net_ranking()`, `calculate_ranking_text()` to populate live scoreboards and post-game screens
- **GameWorld loop** (`Source_Files/GameWorld/marathon2.cpp`): Calls `update_net_game()` each frame to advance game state; calls `player_killed_player()` when damage results in a death
- **Screen rendering** (`Source_Files/RenderOther/screen_drawing.cpp` and `HUDRenderer_Lua.cpp`): Query rankings and compass state for HUD display

### Outgoing (what this file depends on)
- **player.h**: Defines `player_data` structure and `NUMBER_OF_TEAM_COLORS` constant; network_games interrogates player inventory, kills, deaths during ranking calculation
- **GameWorld state**: Indirectly reads monster/player/projectile state during `update_net_game()` to check game-over conditions (kill counts, flag captures, territorial control)
- **Global `team_netgame_parameters`**: Acts as scratch state for game-mode-specific accumulators (team scores, flag holders, king position) that persist across frames

## Design Patterns & Rationale

**Query Pattern with Lazy Formatting**
Functions split into two categories: stat retrieval (`get_player_net_ranking()`) and text formatting (`calculate_ranking_text()`). This decouples scoring logic from display, allowing the HUD to cache or transform rankings without re-computing them.

**Frame-Driven State Machine**
`update_net_game()` is called every frame during the main loop, mirroring the 30 FPS deterministic tick model. This ensures win conditions (kill limits, time limits) are checked at consistent intervals and rankings stay synchronized across networked peers.

**Event Recording with Validation**
`player_killed_player()` returns `bool` to indicate success/failureΓÇöreflecting that not all kill events are valid (e.g., friendly fire in team modes, or invalid player indices). This aligns with the networked multiplayer architecture where stale messages or packet loss can produce spurious kill records.

**Mode-Agnostic Ranking Abstraction**
The `ranking` value returned is opaqueΓÇöits meaning depends on game type. Deathmatch uses kill count, CTF uses flag captures, etc. This avoids an explosion of per-game-mode query functions and lets the implementation change game semantics in `network_games.cpp` without touching this header.

## Data Flow Through This File

```
Incoming:
  player_killed_player() ΓåÉ GameWorld damage ΓåÆ kills recorded in player_data
  update_net_game()     ΓåÉ main game loop (30 FPS tick)
  
Processing:
  update_net_game() reads player_data, checks win conditions, updates team_netgame_parameters
  calculate_player_rankings() scans all players, computes per-player ranking score
  
Outgoing:
  get_player_net_ranking() ΓåÆ HUD scoreboards (live + post-game)
  get_network_compass_state() ΓåÆ overhead map/compass rendering
  game_is_over() ΓåÆ game loop (halt condition)
  get_network_score_text_for_postgame() ΓåÆ end-of-match UI
```

The post-game path is distinct: ranking calculations differ when `game_is_over=true`, suggesting final tallies may apply different rules than live scores.

## Learning Notes

**1990s Multiplayer Design Choices**
- Global `int32 team_netgame_parameters[NUMBER_OF_TEAM_COLORS][2]` is intentionally generic: different game types pack different data into those `[2]` slots (team kill count, flag holder ID, territorial score, etc.). Modern engines would use tagged structs or inheritance; this saves memory and binary size.
- Enum bit flags (`_network_compass_nw | _network_compass_use_beacon`) compress state into a `short`, mirroring the bandwidth-conscious era when replication was expensive.
- The separation of `get_player_net_ranking()` (returns opaque `long`) from `calculate_ranking_text()` (formats it) is good designΓÇögraphics and math are decoupled.

**Networked Multiplayer Invariant**
The file never calls out to network code directly; instead it exposes a deterministic scoring API. This is intentional: `network_games.cpp` implements the logic, but peers are expected to execute it identically, so rankings remain consistent without explicit synchronization. This mirrors the deterministic lockstep model (vs. authoritative server).

## Potential Issues

**Race Condition on `team_netgame_parameters`**
The global `team_netgame_parameters` array is shared mutable state. If `player_killed_player()` or `update_net_game()` can be called concurrently (e.g., if network receive thread and game loop thread are unsynchronized), concurrent access could corrupt team state. The codebase likely avoids this via a single game thread, but there's no lock to guarantee it.

**Post-Game Ranking Divergence**
The `game_is_over` parameter in ranking functions hints that live rankings may differ from final talliesΓÇöe.g., ongoing bonuses/penalties stop when the game ends. This complicates testing and peer consistency; if one peer calculates post-game rankings differently, it could trigger desyncs or disputes over final scores.

**Compass State Encapsulation**
`get_network_compass_state()` returns a bitfield; the enum constants document the bit layout, but there's no accessor API. Callers must manually mask and shift bits, which is error-prone and obscures intent.
