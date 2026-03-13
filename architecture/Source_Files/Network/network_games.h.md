# Source_Files/Network/network_games.h

## File Purpose

Declares the API for managing network multiplayer game state, including initialization, per-frame updates, player/team rankings, scoring, and compass/map UI features for networked matches.

## Core Responsibilities

- Initialize and update network game state and win conditions
- Track and calculate player and team rankings/scores
- Format ranking and scoring text for HUD display
- Determine game-over conditions in networked matches
- Manage network compass state and beacon display for players
- Report kill events (player-on-player damage)
- Query game mode capabilities (scoring, ball physics, etc.)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `player_ranking_data` | struct | Holds a player's index and computed ranking score for display/sorting |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `team_netgame_parameters` | `int32[NUMBER_OF_TEAM_COLORS][2]` | global | Per-team game parameters (e.g., team scores, flags) indexed by color and parameter index |

## Key Functions / Methods

### initialize_net_game
- **Signature:** `void initialize_net_game(void);`
- **Purpose:** Set up network game state at the start of a multiplayer match.
- **Inputs:** None (reads global game mode/parameters)
- **Outputs/Return:** None
- **Side effects:** Initializes global network game state, team parameters
- **Calls:** Not inferable from this file
- **Notes:** Called once per match start

### update_net_game
- **Signature:** `bool update_net_game(void);`
- **Purpose:** Update network game state each frame; determine if game is over.
- **Inputs:** None (reads player data, game state)
- **Outputs/Return:** `true` if the game is over, `false` otherwise
- **Side effects:** Updates rankings, scores, team parameters; may modify game-over flag
- **Calls:** Not inferable from this file
- **Notes:** Called every frame in multiplayer; drives win condition checks

### get_player_net_ranking
- **Signature:** `long get_player_net_ranking(short player_index, short *kills, short *deaths, bool game_is_over);`
- **Purpose:** Retrieve a player's ranking score and extract kill/death counts.
- **Inputs:** `player_index` (player ID), `game_is_over` (affects ranking calculation)
- **Outputs/Return:** Ranking score; outputs kill/death counts via pointers
- **Side effects:** None
- **Calls:** Not inferable from this file
- **Notes:** Ranking meaning depends on game mode (kills, captures, etc.)

### get_team_net_ranking
- **Signature:** `long get_team_net_ranking(short team, short *kills, short *deaths, bool game_is_over);`
- **Purpose:** Retrieve a team's ranking score and aggregate kill/death counts.
- **Inputs:** `team` (team color index), `game_is_over`
- **Outputs/Return:** Team ranking score; outputs aggregate kill/death via pointers
- **Side effects:** None
- **Calls:** Not inferable from this file

### calculate_player_rankings
- **Signature:** `void calculate_player_rankings(struct player_ranking_data *rankings);`
- **Purpose:** Compute and populate ranking array for all players.
- **Inputs:** `rankings` array (caller-allocated)
- **Outputs/Return:** None; fills ranking array
- **Side effects:** None
- **Calls:** Not inferable from this file
- **Notes:** Array size should accommodate all active players

### get_network_compass_state
- **Signature:** `short get_network_compass_state(short player_index);`
- **Purpose:** Query the compass/map display flags for a given player.
- **Inputs:** `player_index`
- **Outputs/Return:** Bitfield combining `_network_compass_*` flags and beacon state
- **Side effects:** None
- **Calls:** Not inferable from this file
- **Notes:** Return value uses `_network_compass_nw/ne/sw/se` (quadrant bits) and `_network_compass_use_beacon`

### game_is_over
- **Signature:** `bool game_is_over(void);`
- **Purpose:** Check if the current network game has reached its end condition.
- **Inputs:** None
- **Outputs/Return:** `true` if match is over, `false` otherwise
- **Side effects:** None
- **Calls:** Not inferable from this file

### player_killed_player
- **Signature:** `bool player_killed_player(short dead_player_index, short aggressor_player_index);`
- **Purpose:** Record a kill event (one player caused another's death).
- **Inputs:** `dead_player_index`, `aggressor_player_index`
- **Outputs/Return:** `true` if the kill was recorded, `false` if invalid
- **Side effects:** Updates damage records, rankings, score state
- **Calls:** Not inferable from this file
- **Notes:** May not succeed if aggressor/victim are invalid or on same team in certain modes

**Other Functions (Helpers)**

- `calculate_ranking_text()`, `calculate_ranking_text_for_post_game()` ΓÇô Format rankings as strings for HUD/scoreboard
- `get_network_score_text_for_postgame()` ΓÇô Format full score summary for post-game screen
- `current_net_game_has_scores()`, `current_game_has_balls()` ΓÇô Query game mode capabilities
- `get_network_joined_message()`, `get_entry_point_flags_for_game_type()` ΓÇô UI text and spawn flags by game type

## Control Flow Notes

This module is invoked during the main game loop:
- **Init phase:** `initialize_net_game()` called once at match start
- **Frame phase:** `update_net_game()` called each frame to advance game state and check win conditions
- **Query phase:** Ranking/score functions queried by HUD/UI code to display live stats
- **Event phase:** `player_killed_player()` called when damage results in a player's death

## External Dependencies

- **Includes:** `player.h` (defines `player_data`, `NUMBER_OF_TEAM_COLORS`)
- **Defined elsewhere:** Player and team data structures, game mode enumeration, damage tracking
