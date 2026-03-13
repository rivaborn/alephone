# Source_Files/Network/Metaserver/metaserver_dialogs.h

## File Purpose
Defines the UI layer for metaserver client interactions in Aleph One. Declares abstract base classes and concrete UI adapters for connecting to the metaserver, browsing games/players, and handling real-time chat and notifications.

## Core Responsibilities
- Define the `MetaserverClientUi` abstract factory for platform-specific UI implementations
- Implement observer pattern adapters (`GlobalMetaserverChatNotificationAdapter`) to receive metaserver events
- Manage UI state for player/game lists, chat entry, and interactive buttons
- Handle user interactions (join game, select player, send chat, mute)
- Coordinate game announcements via `GameAvailableMetaserverAnnouncer`

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GameAvailableMetaserverAnnouncer` | class | Announces an available game to the metaserver |
| `GlobalMetaserverChatNotificationAdapter` | class | Implements `MetaserverClient::NotificationAdapter` to receive and dispatch server events |
| `MetaserverClientUi` | abstract class | Base UI class; uses abstract factory pattern; holds widget pointers and event handlers |

## Global / File-Static State
None.

## Key Functions / Methods

### run_network_metaserver_ui()
- **Signature:** `std::optional<IPaddress> run_network_metaserver_ui();`
- **Purpose:** Entry point to display and run the metaserver client UI
- **Inputs:** None
- **Outputs/Return:** Optional IP address (the server address selected by the user to join)
- **Side effects:** Creates and runs the modal dialog; blocks until user joins or cancels
- **Calls:** Not inferable from this file
- **Notes:** Public API; implementation delegates to `MetaserverClientUi::Create()` and `GetJoinAddressByRunning()`

### setupAndConnectClient(MetaserverClient& client, bool use_remote_hub)
- **Signature:** `void setupAndConnectClient(MetaserverClient& client, bool use_remote_hub);`
- **Purpose:** Configure and establish metaserver connection
- **Inputs:** Reference to `MetaserverClient`; flag for remote hub selection
- **Outputs/Return:** None
- **Side effects:** Calls connection methods on client; establishes network state
- **Calls:** Not inferable from this file
- **Notes:** Comment "This doesn't go here" suggests possible architectural misplacement

### MetaserverClientUi::Create()
- **Signature:** `static std::unique_ptr<MetaserverClientUi> Create();`
- **Purpose:** Abstract factory method; instantiates platform-specific UI subclass
- **Inputs:** None
- **Outputs/Return:** `unique_ptr<MetaserverClientUi>` to the new instance
- **Side effects:** Allocates heap memory; concrete type determined at link-time
- **Calls:** Not inferable (depends on platform)
- **Notes:** Subclasses differ by platform (SDL vs. Carbon/native)

### MetaserverClientUi::GetJoinAddressByRunning()
- **Signature:** `std::optional<IPaddress> GetJoinAddressByRunning();`
- **Purpose:** Run the UI modal and return the user-selected game address
- **Inputs:** None
- **Outputs/Return:** Optional IP address of selected game server
- **Side effects:** Blocks; calls virtual `Run()` which displays dialog
- **Calls:** `Run()` (virtual, subclass-implemented)
- **Notes:** Primary user-facing method for joining a game

### GameAvailableMetaserverAnnouncer::Start(int32 time_limit)
- **Signature:** `void Start(int32 time_limit);`
- **Purpose:** Begin announcing the game to the metaserver
- **Inputs:** Time limit in seconds
- **Outputs/Return:** None
- **Side effects:** Triggers periodic announcements to metaserver via global `gMetaserverClient`
- **Calls:** Not inferable from this file
- **Notes:** Uses global client instance rather than instance member (m_client commented out)

### Event handlers (protected)
- **`GameSelected(GameListMessage::GameListEntry game)`** ΓÇö User selected a game from list
- **`JoinGame(const GameListMessage::GameListEntry&)`** ΓÇö User confirmed join; sends request
- **`PlayerSelected(MetaserverPlayerInfo info)`** ΓÇö User clicked a player name
- **`sendChat()`** ΓÇö Send text from chat entry widget to server
- **`playersInRoomChanged(const std::vector<MetaserverPlayerInfo>&)`** ΓÇö Update player list display
- **`gamesInRoomChanged(const std::vector<GameListMessage::GameListEntry>&)`** ΓÇö Update game list display
- **`UpdatePlayerButtons()`, `UpdateGameButtons()`** ΓÇö Enable/disable buttons based on selection state

## Control Flow Notes
This file forms the UI tier of a three-part architecture: (1) `MetaserverClient` (network and state), (2) `MetaserverClientUi` (UI framework), (3) platform-specific subclasses (SDL/Carbon implementation).

Flow: `run_network_metaserver_ui()` ΓåÆ factory `Create()` ΓåÆ `GetJoinAddressByRunning()` ΓåÆ subclass `Run()` (modal loop) ΓåÆ user selections trigger event handlers ΓåÆ optional returned address to caller.

Notifications flow in reverse: `MetaserverClient` ΓåÆ registered `NotificationAdapter` ΓåÆ `GlobalMetaserverChatNotificationAdapter` ΓåÆ `MetaserverClientUi` method overrides ΓåÆ widget updates.

## External Dependencies
- **`network_metaserver.h`** ΓÇö `MetaserverClient`, `MetaserverClient::NotificationAdapter`, `MetaserverPlayerInfo`, `GameListMessage::GameListEntry`, `RemoteHubServerDescription`, `RoomDescription`, etc.
- **`shared_widgets.h`** ΓÇö `PlayerListWidget`, `GameListWidget`, `EditTextWidget`, `ColorfulChatWidget`, `ButtonWidget`
- **Forward declares:** `game_info` (defined elsewhere), `IPaddress` (defined elsewhere)
- **Implied:** Global instance `gMetaserverClient` referenced by `GameAvailableMetaserverAnnouncer`
