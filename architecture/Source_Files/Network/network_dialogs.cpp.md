# Source_Files/Network/network_dialogs.cpp

## File Purpose
Implements UI dialogs for network multiplayer functionality in Aleph One. Handles game gathering (server), joining (client), setup configuration, LAN discovery, metaserver integration, and postgame statistics display.

## Core Responsibilities
- Provide dialog UIs for network game setup, gathering, and joining phases
- Implement LAN game discovery using SSLP (Service Location Protocol)
- Manage player chat during pregame and metaserver phases
- Handle metaserver connectivity for Internet game listing and remote hub selection
- Support dedicated remote hub (dedicated server) game hosting
- Display postgame carnage reports with dynamic graph selection
- Progress dialog management for long-running network operations
- Preference binding and data synchronization between UI and game state

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `GathererAvailableAnnouncer` | class | Announces game server availability on LAN via SSLP |
| `JoinerSeekingGathererAnnouncer` | class | Searches for available game servers on LAN via SSLP |
| `GatherDialog` | class | Abstract base for server-side gathering UI |
| `JoinDialog` | class | Abstract base for client-side join UI |
| `SetupNetgameDialog` | class | Abstract base for game configuration UI |
| `SdlGatherDialog` | class | SDL concrete implementation of gather dialog |
| `SdlJoinDialog` | class | SDL concrete implementation of join dialog |
| `SdlSetupNetgameDialog` | class | SDL concrete implementation of setup dialog |
| `LevelInt16Pref` | class | Preference binder converting between menu index and level index |
| `TimerInt32Pref` | class | Preference binder for timer values |
| `ColoredChatEntry` | struct | Chat message with sender and type metadata |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gMetaserverClient` | MetaserverClient* | global | Singleton metaserver connection for Internet game listing |
| `gMetaserverChatHistory` | ChatHistory | global | Persists chat messages across metaserver room changes |
| `gPregameChatHistory` | ChatHistory | global | Persists pregame chat messages |
| `sProgressDialog` | dialog* | static | Current progress dialog instance (or NULL) |
| `sProgressMessage` | w_static_text* | static | Message widget in progress dialog |
| `sProgressBar` | w_progress_bar* | static | Progress bar widget in progress dialog |
| `sAdvertiseGameOnMetaserver` | bool | static | Whether current gathering should advertise on metaserver |

## Key Functions / Methods

### network_gather
- **Signature:** `bool network_gather(bool inResumingGame, bool& outUseRemoteHub)`
- **Purpose:** Server entry point; gather players and start networked game
- **Inputs:** `inResumingGame` (true if loading saved netgame); outputs whether remote hub was selected
- **Outputs/Return:** true if game started successfully
- **Side effects:** Creates MetaserverClient if not present; modifies global chat histories; I/O for SSLP and metaserver; network initialization
- **Calls:** `network_game_setup()`, `NetEnter()`, `NetGather()`, `NetGameJoin()`, `GatherDialog::Create()`, `network_gather_remote_hub()`, metaserver functions
- **Notes:** Handles metaserver advertiser lifecycle; cleans up on failure

### network_join
- **Signature:** `int network_join(void)`
- **Purpose:** Client entry point; join an existing network game
- **Inputs:** None (reads from preferences)
- **Outputs/Return:** kNetworkJoined* or kNetworkJoinFailed* code
- **Side effects:** Network initialization; preference I/O; creates/updates MetaserverClient
- **Calls:** `NetEnter()`, `JoinDialog::Create()`, network functions
- **Notes:** Enters preferences on successful join; reads preferences on failure

### network_game_setup
- **Signature:** `bool network_game_setup(player_info*, game_info*, bool ResumingGame, bool& outAdvertiseGameOnMetaserver, bool& outUpnpPortForward, bool& outUseRemoteHub)`
- **Purpose:** Modal dialog for configuring network game parameters
- **Inputs:** Pointers to player/game info structures; resuming game flag
- **Outputs/Return:** true if OK clicked (settings applied); false if cancelled
- **Side effects:** Modifies player/game info; preference read/write; may reload environment
- **Calls:** `SetupNetgameDialog::Create()`, preference functions

### network_gather_remote_hub
- **Signature:** `static uint16 network_gather_remote_hub()`
- **Purpose:** Connect gatherer to dedicated remote hub server with lowest latency
- **Inputs:** None (reads from metaserver client)
- **Outputs/Return:** Remote hub ID (>0) if successful; 0 if failed
- **Side effects:** Creates pinger; pings remote hubs; attempts connection; shows progress dialog and error alerts
- **Calls:** `NetCreatePinger()`, `NetConnectRemoteHub()`, `NetRemovePinger()`, hub_get_minimum_send_period()
- **Notes:** Filters hubs by latency threshold; tries hubs in latency order until one accepts connection

### GatherDialog::GatherNetworkGameByRunning
- **Signature:** `bool GatherNetworkGameByRunning()`
- **Purpose:** Main dialog loop for gathering phase
- **Inputs:** None (reads from UI state and network)
- **Outputs/Return:** true if game started; false if cancelled
- **Side effects:** Sets up callbacks; clears and attaches chat histories; manages autogather state; reads/writes preferences
- **Calls:** `Run()`, `binder.migrate_*()`, chat and callback setup functions
- **Notes:** Creates MetaserverClient if needed and connected; disables metaserver chat choice if not connected

### GatherDialog::idle
- **Signature:** `void idle()`
- **Purpose:** Per-frame update during gathering
- **Inputs:** None (reads from network)
- **Outputs/Return:** None (modifies UI state)
- **Side effects:** Updates player list widget; may trigger game start or stop
- **Calls:** `player_search()`, `gathered_player()`, network state functions, widget updates
- **Notes:** Implements autogather feature; handles remote hub vs. LAN mode differently

### JoinDialog::JoinNetworkGameByRunning
- **Signature:** `const int JoinNetworkGameByRunning()`
- **Purpose:** Main dialog loop for join phase
- **Inputs:** None (reads from UI and preferences)
- **Outputs/Return:** kNetworkJoined* or kNetworkJoinFailed* code
- **Side effects:** Binds preferences to UI widgets; clears chat history; reads preferences
- **Calls:** `Run()`, preference binding functions
- **Notes:** Initializes join_result; disables chat until join succeeds

### JoinDialog::gathererSearch
- **Signature:** `void gathererSearch()`
- **Purpose:** Per-frame search and state update during join phase
- **Inputs:** None (reads from network and SSLP)
- **Outputs/Return:** None (updates UI and join_result)
- **Side effects:** Updates join state; modifies UI (messages, color/team activation); may trigger game start or stop
- **Calls:** `NetUpdateJoinState()`, SSLP functions, network functions, widget callbacks
- **Notes:** Handles metaserver skip-to-first-time logic; bridges from "pre-join" to "post-join" UI state

### display_net_game_stats
- **Signature:** `void display_net_game_stats(void)`
- **Purpose:** Display postgame carnage report with player rankings and graphs
- **Inputs:** None (reads from dynamic_world and topology)
- **Outputs/Return:** None (blocks until user closes dialog)
- **Side effects:** Creates dialog; calculates rankings; announces game deletion to metaserver
- **Calls:** `create_graph_popup_menu()`, `draw_new_graph()`, widget callbacks, dialog run loop
- **Notes:** Graph type determined by selection; switches between player/team/total stats

### open_progress_dialog / close_progress_dialog / draw_progress_bar
- **Signature:** `void open_progress_dialog(size_t message_id, bool show_progress_bar = true)` / similar
- **Purpose:** Manage modeless progress dialogs for long operations
- **Inputs:** Message ID; optional progress (sent/total)
- **Outputs/Return:** None (modifies sProgress* globals)
- **Side effects:** Creates/destroys dialog widgets; redraws screen
- **Calls:** String/widget functions
- **Notes:** Three static widgets persist across calls to avoid flicker; all operations assert valid state

### GathererAvailableAnnouncer
- **Constructor:** Registers game server on LAN via SSLP
- **Destructor:** Unregisters from SSLP
- **pump() [static]:** Processes SSLP housekeeping

### JoinerSeekingGathererAnnouncer
- **Constructor:** Initiates or skips SSLP search based on `shouldSeek` flag
- **Destructor:** Stops SSLP search
- **pump() [static]:** Processes SSLP housekeeping
- **found_gatherer_callback / lost_gatherer_callback [static]:** Redirect SSLP events to NetRetargetJoinAttempts

## Control Flow Notes
- **Gather phase:** `network_gather()` ΓåÆ dialog setup ΓåÆ `GatherDialog::GatherNetworkGameByRunning()` ΓåÆ idle loop ΓåÆ game start or cancel
- **Join phase:** `network_join()` ΓåÆ dialog setup ΓåÆ `JoinDialog::JoinNetworkGameByRunning()` ΓåÆ gatherer search loop ΓåÆ connect or fail
- **Setup phase:** `network_game_setup()` ΓåÆ `SetupNetgameDialog` ΓåÆ user configures ΓåÆ returns to gather/join
- **Postgame:** `display_net_game_stats()` ΓåÆ graph loop ΓåÆ user closes
- **Remote hub:** gather ΓåÆ `network_gather_remote_hub()` ΓåÆ ping selection ΓåÆ hub connect ΓåÆ game proceeds differently
- **Progress:** Long operations wrap with `open_progress_dialog()` / `close_progress_dialog()` and optionally call `draw_progress_bar()`

## External Dependencies
- **Network:** `network.h`, `network_games.h`, `network_messages.h` ΓÇö game state and network protocols
- **Metaserver:** `metaserver_dialogs.h`, MetaserverClient, GameAvailableMetaserverAnnouncer ΓÇö Internet game listing
- **LAN discovery:** `SSLP_API.h` ΓÇö service location protocol
- **UI/Dialog:** `network_dialog_widgets_sdl.h`, `csdialogs.h` ΓÇö widget and dialog framework
- **Game data:** `map.h`, `game_wad.h`, `shell.h`, `TextStrings.h` ΓÇö level info, strings, startup
- **Preferences:** `preferences.h` ΓÇö player/network/graphics config persistence
- **Sound:** `SoundManager.h` ΓÇö dialog sound effects
- **Progress:** `progress.h` ΓÇö progress dialog support
- **Utilities:** `cseries.h` ΓÇö common C series types and macros
