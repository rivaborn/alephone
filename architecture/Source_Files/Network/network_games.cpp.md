# Source_Files/Network/network_games.cpp

## File Purpose
Implements multiplayer game mode logic and scoring for Aleph One's network games. Manages state tracking, ranking calculations, and per-tick game updates for game modes including King of the Hill, Capture the Flag, Tag, Rugby, and Defense.

## Core Responsibilities
- Calculate player and team rankings based on game type and statistics
- Initialize and update game-mode-specific state (beacons, ball carriers, "it" status)
- Determine game-over conditions and format end-game messages
- Compute on-screen compass/beacon directions for gameplay objectives
- Handle per-tick scoring logic (time tracking, goal detection, ball carriers)
- Format ranking displays for HUD and post-game screens

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `player_ranking_data` | struct | Pairs player_index with ranking score for sorting |
| Game parameter enums (`_king_of_hill_time`, `_ball_carrier_time`, etc.) | enum | Indices into `netgame_parameters[2]` arrays for mode-specific stats |
| Compass state enum (`_network_compass_nw`, `_network_compass_use_beacon`) | enum | Bit flags for HUD compass quadrants and beacon mode |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `team_netgame_parameters[NUMBER_OF_TEAM_COLORS][2]` | int32 array | Global | Tracks per-team score/time stats (aggregated across team members) |
| `use_lua_compass`, `lua_compass_beacons`, `lua_compass_states` | extern arrays | Extern (defined in Lua module) | Script-controlled compass override state |

## Key Functions / Methods

### get_player_net_ranking
- **Signature:** `long get_player_net_ranking(short player_index, short *kills, short *deaths, bool game_is_over)`
- **Purpose:** Calculate a player's ranking score based on game type and accumulated statistics.
- **Inputs:** player_index, pointers to receive kill/death counts, game_is_over flag
- **Outputs/Return:** Ranking value (interpretation varies by game type); fills kills/deaths
- **Side effects:** Reads player damage records; may query Lua scoring mode
- **Calls:** `get_player_data()`, `GetLuaScoringMode()`, loops all players for aggregation
- **Notes:** For custom games, delegates to Lua scoring mode. Defense mode compares offender time against defending team. Ranking semantics differ per mode (points, time in seconds, flags pulled, etc.).

### get_team_net_ranking
- **Signature:** `long get_team_net_ranking(short team, short *kills, short *deaths, bool game_is_over)`
- **Purpose:** Aggregate team-wide ranking from team_netgame_parameters and damage records.
- **Inputs:** team index, output pointers, game_is_over
- **Outputs/Return:** Team ranking; fills kills/deaths
- **Side effects:** Reads global team damage and netgame parameter arrays
- **Calls:** Loops all teams for monster damage aggregation; parallel structure to player ranking
- **Notes:** Defense mode calculates defending vs. attacking team rank differently.

### initialize_net_game
- **Signature:** `void initialize_net_game(void)`
- **Purpose:** Set up initial game state for the selected game mode.
- **Inputs:** None (reads GET_GAME_TYPE())
- **Outputs/Return:** None
- **Side effects:** Sets dynamic_world->game_beacon for hill-based modes; sets game_player_index to NONE for tag/rugby/KTMWTB; zeros team_netgame_parameters
- **Calls:** Iterates map polygons to find and average `_polygon_is_hill` centers
- **Notes:** Must be called before first update_net_game(). Hill center computation handles edge case of zero hills.

### get_network_compass_state
- **Signature:** `short get_network_compass_state(short player_index)`
- **Purpose:** Determine which HUD compass quadrants to illuminate based on player position and objective bearing.
- **Inputs:** player_index
- **Outputs/Return:** Compass state flags (quadrants + beacon flag)
- **Side effects:** None
- **Calls:** `get_player_data()`, `get_polygon_data()`, `get_object_data()`, `arctangent()`
- **Notes:** Lua scripts can override compass entirely. Calculates angle to beacon or objective location; sets compass quadrants within NETWORK_COMPASS_SLOP (1/16 circle). Returns all-on if player at objective (in hill for KOTH, is "it" in tag).

### player_killed_player
- **Signature:** `bool player_killed_player(short dead_player_index, short aggressor_player_index)`
- **Purpose:** Handle kill event, with special logic for Tag mode.
- **Inputs:** dead player index, killer index
- **Outputs/Return:** `true` if kill should be attributed; `false` otherwise
- **Side effects:** In Tag: updates game_player_index to dead player (transfer "it"), plays sound
- **Calls:** `get_player_data()`, `play_object_sound()`, `mark_player_network_stats_as_dirty()`
- **Notes:** Currently only implements Tag mode logic. Returns `true` for other modes to allow normal kill attribution.

### update_net_game
- **Signature:** `bool update_net_game(void)`
- **Purpose:** Per-tick game state update: increment score counters, handle ball/flag interactions, update objective tracking.
- **Inputs:** None (reads dynamic_world, GET_GAME_TYPE())
- **Outputs/Return:** Always returns `false` (note: comment says this is ignored; check game_is_over() instead)
- **Side effects:** Increments netgame_parameters and team_netgame_parameters based on player actions; may call `destroy_players_ball()`; updates `game_player_index`; decrements `interface_decay`
- **Calls:** `get_player_data()`, `get_polygon_data()`, `player_has_ball()`, `find_player_ball_color()`, `destroy_players_ball()`, `mark_player_network_stats_as_dirty()`
- **Notes:** Game-mode switch statement. CTF: increment flag pulls when player at own base with enemy flag. KOTH: increment hill time. Tag: increment "it" time. Rugby/KTMWTB: track ball carrier; score on goal. Defense: track offender time in base. Breaks out of loops on certain conditions (assumed one ball).

### calculate_player_rankings
- **Signature:** `void calculate_player_rankings(struct player_ranking_data *rankings)`
- **Purpose:** Sort players by descending ranking into output array.
- **Inputs:** Output array (must have MAXIMUM_NUMBER_OF_PLAYERS slots)
- **Outputs/Return:** Fills rankings array in descending rank order
- **Side effects:** None
- **Calls:** `get_player_net_ranking()` for each player; simple selection sort
- **Notes:** Destructive sort (modifies temporary copy). Selection sort is O(n┬▓).

### calculate_ranking_text
- **Signature:** `void calculate_ranking_text(char *buffer, long ranking)`
- **Purpose:** Format ranking value as human-readable string for live HUD display.
- **Inputs:** buffer (ΓëÑ40 chars), ranking value
- **Outputs/Return:** Fills buffer with formatted string
- **Side effects:** Writes to buffer (sprintf)
- **Calls:** `std::abs()`, `sprintf()`, `GetLuaScoringMode()`
- **Notes:** Most modes: decimal number. Time-based modes: "MM:SS". Percentage for cooperative.

### game_is_over
- **Signature:** `bool game_is_over(void)`
- **Purpose:** Check if game end condition has been reached.
- **Inputs:** None (reads dynamic_world, GET_GAME_TYPE(), GET_GAME_OPTIONS(), Lua end condition)
- **Outputs/Return:** `true` if game should end
- **Side effects:** Disables kill limit flag and sets 2-second grace time on first trigger; may read/modify game_options
- **Calls:** `GetLuaGameEndCondition()`, `get_player_data()`
- **Notes:** Time limit checked first (game_time_remaining). If kill limit enabled, checks mode-specific thresholds: player kills for carnage modes, team flag pulls for CTF, team points for rugby, offender time for defense. Lua can override (no-end / end-now).

### get_entry_point_flags_for_game_type
- **Signature:** `long get_entry_point_flags_for_game_type(size_t game_type)`
- **Purpose:** Map game type to appropriate spawn point flags for level loading.
- **Inputs:** game_type enum value
- **Outputs/Return:** Entry point flag constant (e.g., `_king_of_hill_entry_point`)
- **Side effects:** None
- **Calls:** None
- **Notes:** Used during level load to select player spawn locations. Custom games default to CTF rules.

### player_has_ball (static)
- **Signature:** `static bool player_has_ball(short player_index, short color)`
- **Purpose:** Check whether a player carries a ball item of specified color.
- **Inputs:** player_index, color (SINGLE_BALL_COLOR for rugby/KTMWTB)
- **Outputs/Return:** `true` if player inventory count > 0
- **Side effects:** None
- **Calls:** `get_player_data()`
- **Notes:** Helper for ball-tracking game modes.

## Control Flow Notes
**Per-game-start:** `initialize_net_game()` called once to set up beacons and zero counters.

**Per-tick:** `update_net_game()` increments mode-specific stats (time in hill, time spent "it", flag pulls, etc.), advances ball state machine, and marks UI dirty.

**Per-frame (UI):** Query functions (`get_player_net_ranking()`, `calculate_ranking_text()`, `get_network_compass_state()`) read current state for HUD.

**End-game check:** `game_is_over()` polled by main game loop to trigger end condition. Lua scripts can inject custom end conditions.

**Post-game:** `calculate_player_rankings()` and `calculate_ranking_text_for_post_game()` format final scores.

## External Dependencies
- **map.h:** `dynamic_world`, `map_polygons`, `GET_GAME_TYPE()`, `GET_GAME_OPTIONS()`, `get_polygon_data()`, `_polygon_is_hill`, `_polygon_is_base`
- **player.h:** `player_data`, `current_player_index`, `get_player_data()`, `PLAYER_IS_DEAD()`, `PLAYER_IS_TOTALLY_DEAD()`, team color constants
- **items.h:** `find_player_ball_color()`, `BALL_ITEM_BASE`
- **lua_script.h:** `GetLuaScoringMode()`, `GetLuaGameEndCondition()`, Lua compass state arrays
- **game_window.h:** `mark_player_network_stats_as_dirty()`
- **cseries.h:** `temporary` (string buffer), string formatting macros
- **weapons.c** (external): `destroy_players_ball()` (implemented elsewhere per comment)
- **SoundManager.h:** Sound playback (via `play_object_sound()`)
