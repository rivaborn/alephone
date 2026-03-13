# Source_Files/Network/Metaserver/network_metaserver.h

## File Purpose
Header for the metaserver client that manages connection to a lobby/matchmaking server. Handles login, player/game list synchronization, chat, and game announcements for the Aleph One game engine.

## Core Responsibilities
- Establish and maintain authenticated connection to metaserver
- Synchronize dynamic lists (players, games, rooms) with server updates
- Handle incoming messages (chat, private messages, broadcasts, keep-alive)
- Send outgoing commands (game creation, player state changes, chat)
- Manage player targeting and ignore lists
- Notify application UI of state changes via callback adapter pattern
- Support multiple concurrent client instances via static registry

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MetaserverMaintainedList<tElement>` | template class | Generic container syncing element lists with server; supports add/delete/refresh verbs and target tracking |
| `MetaserverClient` | class | Main metaserver client managing connection, messaging, and state |
| `NotificationAdapter` | nested abstract class | Callback interface for UI notifications (player/game list changes, messages, disconnects) |
| `NotificationAdapterInstaller` | nested class | RAII wrapper for temporarily swapping notification adapters |
| `LoginDeniedException` | nested exception | Thrown on login failure; contains error code enum |
| `ServerConnectException` | nested exception | Thrown on connection failure |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MetaserverClient::s_instances` | `std::set<MetaserverClient*>` | static | Registry of all active metaserver client instances for `pumpAll()` |
| `MetaserverClient::s_ignoreNames` | `std::set<std::string>` | static | Global set of ignored player names across all clients |

## Key Functions / Methods

### connect
- **Signature:** `void connect(const std::string& serverName, uint16 port, const std::string& userName, const std::string& userPassword, bool use_remote_hub)`
- **Purpose:** Establish authenticated connection to metaserver
- **Inputs:** Server hostname/IP, port, login credentials, remote hub flag
- **Outputs/Return:** None (throws on failure)
- **Side effects:** Creates network channel, initializer, dispatcher; registers message handlers; logs in
- **Calls:** Implicitly constructs internal message handling pipeline
- **Notes:** Throws `ServerConnectException` or `LoginDeniedException`; blocks until login completes or fails

### disconnect
- **Signature:** `void disconnect()`
- **Purpose:** Close metaserver connection gracefully
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Closes communication channel; clears internal state
- **Calls:** Destructs `m_channel`, `m_inflater`, `m_dispatcher`
- **Notes:** Safe to call multiple times

### pump
- **Signature:** `void pump()`
- **Purpose:** Process one iteration of incoming messages for this client
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Reads from network channel, dispatches messages, invokes notification adapter callbacks
- **Calls:** Message dispatcher
- **Notes:** Non-blocking; should be called regularly in main loop

### pumpAll
- **Signature:** `static void pumpAll()`
- **Purpose:** Pump all registered metaserver client instances in one call
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Iterates `s_instances`, calls `pump()` on each
- **Calls:** `pump()` on each instance
- **Notes:** Convenience for applications with multiple metaserver clients

### setPlayerName, setAway, setMode, setPlayerTeamName
- **Signature:** `void setPlayerName(const std::string& name)`, `void setAway(bool away, const std::string& away_message)`, etc.
- **Purpose:** Update local player state and broadcast to server
- **Inputs:** New name, away status/message, mode flags, team name
- **Outputs/Return:** None
- **Side effects:** Updates `m_playerName`, `m_teamName`; sends `NameAndTeamMessage` or `SetPlayerModeMessage`
- **Calls:** Direct network write
- **Notes:** Away message and mode changes may have special handling on server

### sendChatMessage, sendPrivateMessage
- **Signature:** `void sendChatMessage(const std::string& message)`, `void sendPrivateMessage(MetaserverPlayerInfo::IDType destination, const std::string& message)`
- **Purpose:** Send chat or directed private message to server
- **Inputs:** Message text; destination player ID for private messages
- **Outputs/Return:** None
- **Side effects:** Constructs and sends message object
- **Calls:** Network writer
- **Notes:** Server echoes chat to all; private messages routed by ID

### announceGame, announcePlayersInGame, announceGameStarted, announceGameReset, announceGameDeleted
- **Signature:** `void announceGame(uint16 gamePort, const GameDescription& description, uint16 remoteHubId)`, etc.
- **Purpose:** Notify server of local game state lifecycle (creation, player count, start, reset, deletion)
- **Inputs:** Game port, description struct, player count, game time, remote hub ID
- **Outputs/Return:** None
- **Side effects:** Stores `m_gameDescription`, `m_gamePort`, `m_remoteHubId`, `m_gameAnnounced` flags; sends messages
- **Calls:** Various message constructors and network writes
- **Notes:** `announceGame()` stores state; others use cached state; `announceGameStarted()` takes game duration in seconds

### gamesInRoomUpdate
- **Signature:** `const std::vector<GameListMessage::GameListEntry> gamesInRoomUpdate(bool reset_ping)`
- **Purpose:** Retrieve current game list and optionally reset ping timers
- **Inputs:** Flag to reset ping tracking
- **Outputs/Return:** Vector of game entries
- **Side effects:** May clear `ping_games` map if `reset_ping` is true
- **Calls:** `m_gamesInRoom.entries()`
- **Notes:** Used for UI refresh; ping tracking appears incomplete

### ignore, is_ignored
- **Signature:** `void ignore(const std::string& name)`, `void ignore(MetaserverPlayerInfo::IDType id)`, `bool is_ignored(MetaserverPlayerInfo::IDType id)`
- **Purpose:** Add/remove players from ignore list; check ignore status
- **Inputs:** Player name or ID
- **Outputs/Return:** Boolean for query
- **Side effects:** Modifies static `s_ignoreNames` or internal ignore tracking
- **Calls:** Set operations
- **Notes:** Name-based ignores are static across all clients

### player_target, game_target, find_player, find_game
- **Signature:** `void player_target(IDType id)`, `IDType player_target()`, `const MetaserverPlayerInfo* find_player(IDType id)`, etc.
- **Purpose:** Manage/query "target" (selected/focused) player or game for UI highlighting
- **Inputs:** ID to target, or none for getter
- **Outputs/Return:** ID or pointer (null if not found)
- **Side effects:** Updates `m_playersInRoom` or `m_gamesInRoom` target state
- **Calls:** Template list operations
- **Notes:** Target updates propagated to list entries via `target(bool)` method

### rooms, setRoom, playersInRoom, gamesInRoom
- **Signature:** Various getters and setters for room/list data
- **Purpose:** Query current room list and game/player state; switch rooms
- **Inputs:** Room description for `setRoom()`
- **Outputs/Return:** Rooms vector or game/player vectors
- **Side effects:** `setRoom()` updates `m_room`
- **Calls:** List enumeration methods
- **Notes:** `setRoom()` sends room login message

## Control Flow Notes

**Initialization:** `connect()` creates the network pipeline (channel ΓåÆ inflater ΓåÆ dispatcher) and registers message handlers for each message type (room list, player list, chat, etc.). Login is synchronous.

**Main loop:** Periodically call `pump()` or `pumpAll()` to receive and dispatch messages. Message handlers update internal lists and invoke notification adapter callbacks.

**State changes:** User calls `setPlayerName()`, `announceGame()`, etc., which send messages directly. Server responses (player list updates, game list changes) flow through message handlers and trigger notifications.

**Shutdown:** `disconnect()` or destructor cleans up resources.

## External Dependencies

- **Includes:** `metaserver_messages.h` (message types and `RoomDescription`), `Logging.h` (macros: `logAnomaly()`), `exception`, `vector`, `map`, `memory`, `set`, `stdexcept`
- **Defined elsewhere (forward declared or external):** 
  - `CommunicationsChannel`, `MessageInflater`, `MessageHandler`, `MessageDispatcher` (network pipeline)
  - `Message`, `ChatMessage`, `BroadcastMessage`, `PrivateMessage`, `PlayerListMessage`, `RoomListMessage`, `GameListMessage`, `RemoteHubListMessage`, `SetPlayerDataMessage` (message types)
  - `MetaserverPlayerInfo`, `GameListMessage::GameListEntry` (data types from messages.h)
  - `GameDescription`, `RemoteHubServerDescription` (from messages.h)
  - Standard library types (`uint8`, `uint16`, `uint32`, etc. ΓÇö likely typedefs)
