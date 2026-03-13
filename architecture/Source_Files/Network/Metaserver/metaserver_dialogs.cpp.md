# Source_Files/Network/Metaserver/metaserver_dialogs.cpp

## File Purpose

Implements the metaserver client UI dialog for Aleph One. Provides the presentation layer for game discovery, joining, and in-game chat. Handles user interactions such as selecting games/players, sending messages, and joining hosted games discovered via the metaserver.

## Core Responsibilities

- **Game Discovery UI**: Displays available games and players in a room; manages game/player selection state
- **Dialog Lifecycle**: Runs modal dialog with widget callbacks; produces a join address on successful game selection
- **Game Announcement**: Packages local game info (map, difficulty, options) and announces to metaserver
- **Chat Integration**: Bridges metaserver notifications (join/leave, messages) to global chat history; plays UI sounds
- **Update Checking**: Prompts user if a new build is available before connecting (non-Steam only)
- **Notification Routing**: Adapter pattern to receive and process metaserver events (player list, game list, chat, disconnection)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `GameAvailableMetaserverAnnouncer` | class | Announces a hosted game; packages game description and sends to metaserver on Start() |
| `GlobalMetaserverChatNotificationAdapter` | class | Implements `MetaserverClient::NotificationAdapter`; appends notifications to global chat history |
| `MetaserverClientUi` | class | Abstract UI dialog; manages widgets, handles user selection/input, returns join address |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gMetaserverChatHistory` | `ChatHistory` | extern (global) | Holds all chat messages and notifications displayed in the dialog |
| `gMetaserverClient` | `MetaserverClient*` | extern (global) | Singleton metaserver connection; target of all notifications and commands |
| `user_informed` (setupAndConnectClient) | `static bool` | static | One-shot flag; prevents repeated update dialogs across reconnects |
| `first_check` (setupAndConnectClient) | `static bool` | static | One-shot flag; defers initial update check to avoid UI blocking |

## Key Functions / Methods

### run_network_metaserver_ui
- **Signature:** `std::optional<IPaddress> run_network_metaserver_ui()`
- **Purpose:** Entry point; creates a MetaserverClientUi and runs it to completion
- **Inputs:** None
- **Outputs/Return:** `std::optional<IPaddress>` ΓÇö address + port of the selected game, or nullopt if cancelled
- **Side effects:** Blocks until dialog closes; modifies global `gMetaserverClient` and `gMetaserverChatHistory`
- **Calls:** `MetaserverClientUi::Create()`, `GetJoinAddressByRunning()`
- **Notes:** Facade function; hides concrete UI implementation chosen at link-time

### setupAndConnectClient
- **Signature:** `void setupAndConnectClient(MetaserverClient& client, bool use_remote_hub)`
- **Purpose:** Configures client with player name; checks for updates; connects to metaserver
- **Inputs:** `client` (MetaserverClient reference), `use_remote_hub` (bool)
- **Outputs/Return:** None (void)
- **Side effects:** Sets client player name; shows update prompt (non-Steam); blocks on update check; connects client to `A1_METASERVER_HOST:6321`
- **Calls:** `Update::instance()->GetStatus()`, `open_progress_dialog()`, `close_progress_dialog()`, `client.setPlayerName()`, `client.connect()`
- **Notes:** Uses static flags to avoid repeated update checks; waits up to 2.5s for update status before prompting user

### GameAvailableMetaserverAnnouncer (constructor)
- **Signature:** `GameAvailableMetaserverAnnouncer(const game_info& info, uint16 remote_hub_id = 0)`
- **Purpose:** Packages local game metadata and announces it to the metaserver
- **Inputs:** `info` (game_info struct), `remote_hub_id` (optional)
- **Outputs/Return:** None (constructor)
- **Side effects:** Calls `gMetaserverClient->announceGame()` with a `GameDescription` built from game_info
- **Calls:** `level_has_embedded_physics_lua()`, `gMetaserverClient->announceGame()`
- **Notes:** Detects embedded Lua/physics; builds description string from player name and level; converts untimed games (>7 days) to -1

### GameAvailableMetaserverAnnouncer::Start
- **Signature:** `void Start(int32 time_limit)`
- **Purpose:** Notifies metaserver that the game has started
- **Inputs:** `time_limit` (remaining game time in seconds)
- **Outputs/Return:** None
- **Side effects:** Calls `gMetaserverClient->announceGameStarted()`
- **Calls:** `gMetaserverClient->announceGameStarted()`

### MetaserverClientUi::GetJoinAddressByRunning
- **Signature:** `std::optional<IPaddress> GetJoinAddressByRunning()`
- **Purpose:** Main dialog loop; processes callbacks until user selects a game or cancels
- **Inputs:** None
- **Outputs/Return:** `std::optional<IPaddress>` ΓÇö join address if successful, nullopt if cancelled
- **Side effects:** Connects client; registers notification adapter; clears and populates chat history; runs modal dialog; may delete/recreate global `gMetaserverClient`
- **Calls:** `setupAndConnectClient()`, `associateNotificationAdapter()`, `std::bind()` for all widget callbacks, `Run()`, `handleCancel()`
- **Notes:** One-shot method; asserts if called twice on same instance. Sets up all widget callbacks via std::bind

### MetaserverClientUi::GameSelected
- **Signature:** `void GameSelected(GameListMessage::GameListEntry game)`
- **Purpose:** Handles game list selection; double-click to join (if scenario compatible and not running)
- **Inputs:** `game` (selected game entry)
- **Outputs/Return:** None
- **Side effects:** Sets/clears `game_target`; calls `JoinClicked()` on double-click; re-sorts and updates game list widget
- **Calls:** `machine_tick_count()`, `Scenario::instance()->IsCompatible()`, `JoinClicked()`, `UpdateGameButtons()`

### MetaserverClientUi::PlayerSelected
- **Signature:** `void PlayerSelected(MetaserverPlayerInfo info)`
- **Purpose:** Handles player list selection; toggles target; remembers Ctrl state for sticky selection
- **Inputs:** `info` (selected player info)
- **Outputs/Return:** None
- **Side effects:** Toggles `player_target`; sets `m_stay_selected` based on Ctrl key; updates and re-sorts player list
- **Calls:** `SDL_GetModState()`, `UpdatePlayerButtons()`

### GlobalMetaserverChatNotificationAdapter::playersInRoomChanged
- **Signature:** `void playersInRoomChanged(const std::vector<MetaserverPlayerInfo>& playerChanges)`
- **Purpose:** Notifies chat when players join/leave; formats and appends messages
- **Inputs:** `playerChanges` (vector of player info with verb: kAdd or kDelete)
- **Outputs/Return:** None
- **Side effects:** Appends "Player has joined/left the room" to chat history via `receivedLocalMessage()`
- **Calls:** `receivedLocalMessage()`

**Notes on remaining notify/chat methods** (`gamesInRoomChanged`, `receivedChatMessage`, `receivedPrivateMessage`, `receivedBroadcastMessage`, `receivedLocalMessage`, `roomDisconnected`): All follow a similar pattern ΓÇö construct a `ColoredChatEntry`, optionally look up player color, append to `gMetaserverChatHistory`, and play a sound.

## Control Flow Notes

1. **Initialization ΓåÆ Lobby:**
   - `run_network_metaserver_ui()` ΓåÆ `MetaserverClientUi::Create()` ΓåÆ `GetJoinAddressByRunning()`
   - `setupAndConnectClient()` checks for updates, connects to metaserver
   - Notification adapter registered; chat cleared

2. **User Interaction (lobby phase):**
   - Widget callbacks (`GameSelected`, `PlayerSelected`, `ChatTextEntered`) update UI state and send commands via `gMetaserverClient`
   - Metaserver notifies of player/game changes ΓåÆ adapter appends to chat history

3. **Game Join:**
   - `GameSelected()` (double-click) ΓåÆ `JoinClicked()` ΓåÆ `JoinGame()` ΓåÆ sets `m_joinAddress` ΓåÆ `Stop()` ΓåÆ return to caller

4. **Cleanup:**
   - `handleCancel()` recreates global `gMetaserverClient` (clean slate), calls `Cancel()`
   - `delete_widgets()` frees all widget pointers

## External Dependencies

- **Network & Metaserver:** `MetaserverClient`, `MetaserverPlayerInfo`, `GameListMessage::GameListEntry`, `IPaddress` (network_metaserver.h)
- **UI Framework:** `dialog`, `w_title`, `w_spacer`, `w_static_text`, `w_hyperlink`, `w_button`, widget classes from implied header (csdialogs.h via cseries.h)
- **Game State:** `Scenario::instance()`, `game_info` struct, physics/Lua detection (`level_has_embedded_physics_lua`)
- **Preferences:** `player_preferences`, `network_preferences`, `environment_preferences` (preferences.h)
- **Update System:** `Update::instance()`, update check dialogs (Update.h, progress.h)
- **Audio:** `PlayInterfaceButtonSound()` (SoundManager.h)
- **Chat Widget:** `ColorfulChatWidget`, `ChatHistory`, `EditTextWidget`, `PlayerListWidget`, `GameListWidget`, `ButtonWidget` (shared_widgets.h)

Not defined here: `MetaserverClient` internals, widget implementations, chat history storage.
