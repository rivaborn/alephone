# Source_Files/Network/network_dialogs.h

## File Purpose
Header file declaring network game UI dialogs for Aleph One. Defines classes for hosting/gathering players, joining games, and configuring network game parameters, plus carnage report (postgame statistics) functionality.

## Core Responsibilities
- Abstract dialog classes for network game setup workflow (`GatherDialog`, `JoinDialog`, `SetupNetgameDialog`)
- Dialog control IDs, string resource IDs, and game configuration enums
- Player ranking structure and postgame carnage report declarations
- Factory methods for platform-specific dialog implementations (determined at link-time)
- Chat and network callback interfaces for dialogs

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `net_rank` | struct | Stores player kill/death/score data and ranking metadata |
| `GatherDialog` | class | Abstract base for host/gatherer dialog UI |
| `JoinDialog` | class | Abstract base for joiner dialog UI |
| `SetupNetgameDialog` | class | Abstract base for game setup configuration dialog |
| `GatherCallbacks` | class (from network.h) | Interface for network event callbacks during gathering |
| `ChatCallbacks` | class (from network.h) | Interface for chat message handling |
| `GlobalMetaserverChatNotificationAdapter` | class (from metaserver_dialogs.h) | Adapter for metaserver chat notifications |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rankings` | `net_rank[MAXIMUM_NUMBER_OF_PLAYERS]` | global | Player rankings for postgame report display |

## Key Functions / Methods

### GatherDialog::Create
- Signature: `static std::unique_ptr<GatherDialog> Create(bool remote_hub_mode)`
- Purpose: Abstract factory; returns platform-specific implementation
- Inputs: `remote_hub_mode` ΓÇö whether to use remote hub
- Outputs/Return: Unique pointer to concrete `GatherDialog` subclass
- Side effects: None (factory method)
- Calls: Determined at link-time
- Notes: Caller owns returned pointer lifetime

### GatherDialog::GatherNetworkGameByRunning
- Signature: `bool GatherNetworkGameByRunning()`
- Purpose: Main event loop for host dialog; waits for players to join and game start
- Inputs: None (dialog state managed internally)
- Outputs/Return: Boolean success/failure
- Side effects: Blocks until user cancels or starts game; updates `m_ungathered_players`, calls network callbacks
- Calls: `Run()` (abstract), network/chat methods
- Notes: Entry point after `SetupNetworkGameByRunning()` completes

### GatherDialog network callbacks (virtual)
- **`JoiningPlayerArrived`** ΓÇö A prospective joiner discovered
- **`JoinSucceeded`** ΓÇö Joiner accepted into the game
- **`JoiningPlayerDropped`** ΓÇö Prospective joiner left/timed out (returns success/fail)
- **`JoinedPlayerDropped`** ΓÇö Accepted player disconnected (returns success/fail)
- **`JoinedPlayerChanged`** ΓÇö Accepted player changed team/color

### JoinDialog::Create
- Signature: `static std::unique_ptr<JoinDialog> Create()`
- Purpose: Abstract factory for joiner dialog
- Outputs/Return: Unique pointer to concrete `JoinDialog` subclass
- Side effects: None

### JoinDialog::JoinNetworkGameByRunning
- Signature: `const int JoinNetworkGameByRunning()`
- Purpose: Main event loop for joiner; searches for gatherer and waits for game start
- Outputs/Return: Integer result code (0 = success)
- Side effects: Blocks; updates chat widgets; calls `gathererSearch()`, `attemptJoin()`
- Calls: `Run()` (abstract), search/join methods

### JoinDialog helper methods
- **`gathererSearch()`** ΓÇö Locate available hosts via network announcement
- **`attemptJoin()`** ΓÇö Send join request to selected gatherer
- **`getJoinAddressFromMetaserver()`** ΓÇö Query metaserver for game address
- **`changeColours()`** ΓÇö Change player team/color preference

### SetupNetgameDialog::Create
- Signature: `static std::unique_ptr<SetupNetgameDialog> Create()`
- Purpose: Abstract factory for game setup dialog
- Outputs/Return: Unique pointer to concrete `SetupNetgameDialog` subclass

### SetupNetgameDialog::SetupNetworkGameByRunning
- Signature: `bool SetupNetworkGameByRunning(player_info*, game_info*, bool ResumingGame, bool& outAdvertiseGameOnMetaserver, bool& outUpnpPortForward, bool& outUseRemoteHub)`
- Purpose: Configure game parameters (level, difficulty, time/kill limits, teams, script, metaserver options)
- Inputs: Player data, game data, resuming flag, output refs for metaserver/UPnP/hub settings
- Outputs/Return: Boolean success; output refs filled with user's configuration choices
- Side effects: Modifies `*player_information`, `*game_information`; calls `Run()`, event handlers
- Calls: `informationIsAcceptable()`, game type setup methods, validation
- Notes: Entry point in network game initialization workflow

### SetupNetgameDialog game configuration methods
- **`setupForUntimedGame()`**, **`setupForTimedGame()`**, **`setupForScoreGame()`** ΓÇö Configure UI based on end condition type
- **`gameTypeHit()`** ΓÇö Handle game type dropdown change
- **`limitTypeHit()`** ΓÇö Handle time/kill/untimed switch
- **`chooseMapHit()`** ΓÇö Handle map selection
- **`teamsHit()`** ΓÇö Handle teams toggle
- **`okHit()`** ΓÇö Validate and accept settings

### Postgame carnage report functions (extern)
- **`calculate_rankings(net_rank*, short num_players)`** ΓÇö Compute kill/death rankings
- **`rank_compare(const void*, const void*)`** ΓÇö qsort-compatible comparator for player rankings
- **`team_rank_compare()`, `score_rank_compare()`** ΓÇö Comparators for team and score rankings
- **`find_graph_mode(NetgameOutcomeData&, short* index)`** ΓÇö Determine which graph to display
- **`draw_new_graph()`, `draw_player_graph()`, `draw_totals_graph()`, `draw_team_totals_graph()`, `draw_total_scores_graph()`, `draw_team_total_scores_graph()`** ΓÇö Render different postgame statistics visualizations
- **`draw_names()`, `draw_kill_bars()`, `draw_score_bars()`** ΓÇö Draw specific elements of carnage report
- **`update_carnage_summary()`** ΓÇö Compute and cache summary statistics
- **`calculate_max_kills(size_t)`** ΓÇö Determine y-axis scale for kill graphs
- **`get_net_color(short index, RGBColor*)`** ΓÇö Get team/player color for rendering

### Level selection helpers (extern)
- **`menu_index_to_level_entry()`, `menu_index_to_level_index()`, `level_index_to_menu_index()`** ΓÇö Convert between menu indices and level descriptors (used by SetupNetgameDialog)

## Control Flow Notes
**Network game initialization sequence:**
1. Host calls `SetupNetgameDialog::SetupNetworkGameByRunning()` (configure level, rules, teams)
2. On success, calls `GatherDialog::GatherNetworkGameByRunning()` (wait for joiners)
3. Joiners call `JoinDialog::JoinNetworkGameByRunning()` (search, join, wait for start)
4. Network layer triggers callbacks (`JoiningPlayerArrived`, `JoinSucceeded`, etc.)
5. Game runs; on completion, postgame report functions render carnage statistics

**Dialog architecture:**
- Three abstract base classes with `Create()` factory methods
- Concrete implementations (SDL/Mac) determined at link-time
- All inherit from `GlobalMetaserverChatNotificationAdapter` to receive metaserver chat
- Protected `Run()` and `Stop()` methods called by public entry points; subclasses implement platform-specific UI

## External Dependencies
- `player.h` ΓÇö `MAXIMUM_NUMBER_OF_PLAYERS`, `player_info`
- `network.h` ΓÇö `game_info`, `player_info`, `prospective_joiner_info`, `GatherCallbacks`, `ChatCallbacks`
- `network_private.h` ΓÇö `JoinerSeekingGathererAnnouncer`
- `FileHandler.h` ΓÇö `FileSpecifier`
- `network_metaserver.h` ΓÇö `MetaserverClient`
- `metaserver_dialogs.h` ΓÇö `GlobalMetaserverChatNotificationAdapter`, `GameAvailableMetaserverAnnouncer`
- `shared_widgets.h` ΓÇö `ButtonWidget`, `EditTextWidget`, `SelectorWidget`, `PlayersInGameWidget`, `EditNumberWidget`, `ToggleWidget`, `EnvSelectWidget`, `ColorfulChatWidget`, etc.
- `preferences_widgets_sdl.h` ΓÇö SDL widget types
- Standard library: `<string>`, `<map>`, `<set>`
