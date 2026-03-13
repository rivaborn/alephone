# Source_Files/Network/Metaserver/network_metaserver.cpp

## File Purpose
Implements the `MetaserverClient` class, a client for connecting to a game metaserver (Aleph One). Handles authentication, room/player/game listing, chat messaging, and game lifecycle announcements. Core networking logic for multiplayer game discovery and communication.

## Core Responsibilities
- Establish and maintain TCP connections to metaserver and room servers
- Handle authentication (supports plaintext, XOR encryption, HTTPS key exchange)
- Process incoming messages (chat, broadcast, player list updates, game list, etc.)
- Queue outgoing messages (chat, game announcements, status updates)
- Manage player ignore lists with filtering at message-handler level
- Announce games to metaserver and track player count / game state changes
- Ping game servers asynchronously to measure latency
- Route notifications to UI layer via `NotificationAdapter` callback interface

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MetaserverClient::NotificationAdapter` | interface (inner class) | Callback contract for UI updates on message/player/game changes |
| `MetaserverClient::NotificationAdapterInstaller` | RAII class (inner) | Scope-guarded temporary notification adapter swapping |
| `MetaserverMaintainedList<T>` | template class (in header) | Generic list maintaining add/delete/refresh semantics for players/games |
| `LoginDeniedException` | exception class (inner) | Codes for login rejection (invalid version, bad password, etc.) |
| `ServerConnectException` | exception class (inner) | Generic connection/protocol failures |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_instances` | `std::set<MetaserverClient*>` | static class member | All active client instances (used for `pumpAll()`) |
| `s_ignoreNames` | `std::set<std::string>` | static class member | Global ignore list; "Bacon" pre-added |
| `kKeyLength` | const int (16) | file-static | Encryption key buffer size |

## Key Functions / Methods

### connect
- **Signature:** `void connect(const std::string& serverName, uint16 port, const std::string& userName, const std::string& userPassword, bool use_remote_hub)`
- **Purpose:** Establish metaserver connection, authenticate, fetch room/hub lists, then disconnect and reconnect to room server.
- **Inputs:** Server hostname/port, credentials, optional remote hub flag
- **Outputs/Return:** None (throws on failure)
- **Side effects:** 
  - Clears player/game lists
  - Modifies `m_channel`, `m_playerID`, `m_rooms`, `m_remoteHubServers`
  - Switches message handler from `m_loginDispatcher` to `m_dispatcher`
  - I/O: TCP handshakes, HTTPS POST for key derivation (if configured)
- **Calls:** 
  - `CommunicationsChannel::connect()`, `receiveMessage()`, `receiveSpecificMessageOrThrow()`
  - `HTTPClient::Post()` for HTTPS encryption variant
  - `MessageDispatcher::handle()` for room/hub list processing
- **Notes:** 
  - Supports three encryption types: plaintext, "braindead simple" (XOR then bitwise ops), and HTTPS via POST
  - Sequence: metaserver login ΓåÆ key exchange ΓåÆ room list ΓåÆ (optional) remote hub list ΓåÆ disconnect metaserver ΓåÆ room server login ΓåÆ room acceptance
  - Throws `ServerConnectException`, `LoginDeniedException`, or `CommunicationsChannel::FailedToReceiveSpecificMessageException`

### pump
- **Signature:** `void pump()`
- **Purpose:** Process one iteration of network I/O and dispatch incoming messages.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** 
  - Calls `CommunicationsChannel::pump()` (TCP recv/send)
  - Dispatches queued messages to handlers
  - Invokes `m_notificationAdapter->roomDisconnected()` on first detection of disconnect (if adapter set)
- **Calls:** `m_channel->pump()`, `m_channel->dispatchIncomingMessages()`
- **Notes:** Non-blocking; tracks disconnect notification to avoid duplicate callbacks

### sendChatMessage
- **Signature:** `void sendChatMessage(const std::string& message)`
- **Purpose:** Send chat message or execute special server commands (`.available`, `.who`, `.ignore`, `.games`, `.pork`).
- **Inputs:** Message string
- **Outputs/Return:** None
- **Side effects:** 
  - Queues `ChatMessage` to channel for normal messages
  - May invoke `m_notificationAdapter->receivedLocalMessage()` for command responses (filtered player lists, ignore list, games)
- **Calls:** `m_channel->enqueueOutgoingMessage()`
- **Notes:** 
  - `.available` / `.avail`: lists non-away players
  - `.who`: lists all players
  - `.ignore`: shows current ignore list; `.ignore <name>` toggles ignore
  - `.games`: displays running games
  - `.pork`: replies "NO BACON FOR YOU" (hardcoded Easter egg; ignores if message starts with ".pork")

### gamesInRoomUpdate
- **Signature:** `const std::vector<GameListMessage::GameListEntry> gamesInRoomUpdate(bool reset_ping)`
- **Purpose:** Ping all non-running games to measure latency; return updated games with new ping times.
- **Inputs:** `reset_ping`: if true, destroy and recreate pinger; if false, skip already-pinged games
- **Outputs/Return:** Vector of games with updated latencies
- **Side effects:** 
  - Updates `m_gamesInRoom` via `processUpdates()` for changed latencies
  - Calls `NetRemovePinger()`, `NetCreatePinger()`, registers IPs with pinger, calls `pinger->Ping()`
  - Modifies `ping_games` map (game ID ΓåÆ pinger handle)
- **Calls:** `NetGetPinger()`, `NetRemovePinger()`, `NetCreatePinger()`, pinger methods
- **Notes:** Only pings games marked `!running()`; skips already-pinged games unless `reset_ping` is true. Verb is hardcoded to 2 (kRefresh).

### handlePlayerListMessage
- **Signature:** `void handlePlayerListMessage(PlayerListMessage* inMessage, CommunicationsChannel* inChannel)`
- **Purpose:** Process server's player list update (add/delete/refresh players).
- **Inputs:** Incoming player list message, channel reference
- **Outputs/Return:** None
- **Side effects:** 
  - Removes "invisible" (already-deleted) players from update before processing
  - Updates `m_playersInRoom` list
  - Notifies adapter of player changes
- **Calls:** `m_playersInRoom.processUpdates()`, `m_notificationAdapter->playersInRoomChanged()`
- **Notes:** Filters out delete notifications for players not in the local list (to handle server-side invisibility quirk)

### handleChatMessage / handlePrivateMessage
- **Signature:** `void handleChatMessage(ChatMessage* message, CommunicationsChannel* inChannel)` and `handlePrivateMessage` variant
- **Purpose:** Process incoming chat/private messages, apply filtering, route to notification adapter.
- **Inputs:** Message, channel reference
- **Outputs/Return:** None
- **Side effects:** 
  - Parses sender name, strips formatting codes and special prefix `\260`
  - Checks `network_preferences->mute_metaserver_guests` to filter "Guest *" names
  - Checks `s_ignoreNames` to filter ignored players
  - Invokes adapter callbacks
- **Calls:** `remove_formatting()`, `m_notificationAdapter->receivedChatMessage()` / `receivedPrivateMessage()`
- **Notes:** 
  - Only processes if `internalType() == 0` and adapter exists
  - Both methods have identical logic; `PrivateMessage` handler is separate but mirrors chat logic
  - Formatting removal uses `|` followed by style codes (p/b/i/l/r/c/s)

### announceGame / announcePlayersInGame / announceGameStarted / announceGameReset / announceGameDeleted
- **Signature:** Various: `void announceGame(uint16 gamePort, const GameDescription& description, uint16 remoteHubId)`, etc.
- **Purpose:** Announce game server to metaserver, update player counts, mark game as started/reset, or remove game.
- **Inputs:** Game port, description (includes num players, closed, running flags), remote hub ID
- **Outputs/Return:** None
- **Side effects:** 
  - Queues `CreateGameMessage` or `StartGameMessage` to channel
  - Updates `m_gameDescription`, `m_gamePort`, `m_remoteHubId` state
  - Sets `m_gameAnnounced` flag
- **Calls:** `m_channel->enqueueOutgoingMessage()`
- **Notes:** 
  - `announceGameStarted()` converts `INT32_MAX` to `INT32_MIN` (likely to indicate infinity/no limit)
  - `announceGameDeleted()` is idempotent (checks `m_gameAnnounced` flag)

### ignore / is_ignored
- **Signature:** `void ignore(const std::string& name)`, `void ignore(MetaserverPlayerInfo::IDType id)`, `bool is_ignored(MetaserverPlayerInfo::IDType id)`
- **Purpose:** Toggle ignore status for a player by name or ID; check if a player is ignored.
- **Inputs:** Player name or ID
- **Outputs/Return:** Boolean for `is_ignored()`
- **Side effects:** 
  - Adds/removes from global `s_ignoreNames` set
  - Invokes adapter callbacks for UI feedback
  - Strips formatting codes and `\260` prefix before storing
- **Calls:** `remove_formatting()`, `m_playersInRoom.find()`, `m_notificationAdapter->receivedLocalMessage()`
- **Notes:** 
  - Toggline: if already in list, removes; otherwise adds
  - Used to filter chat/private messages at handler level

### remove_formatting (static)
- **Signature:** `static std::string remove_formatting(const std::string &s)`
- **Purpose:** Strip Aleph One style codes (`|p`, `|b`, `|i`, `|l`, `|r`, `|c`, `|s`) from text.
- **Inputs:** String with potential formatting
- **Outputs/Return:** Cleaned string
- **Calls:** `style_code()`
- **Notes:** Iterates through string, skips `|` + style-code pairs

## Control Flow Notes
**Connection phase:** 
- Constructor initializes handlers and message prototypes
- `connect()` performs authentication handshake (metaserver ΓåÆ room server)
- Handlers registered with `m_dispatcher` for room messages, `m_loginDispatcher` for login phase

**Message pump loop:** 
- `pump()` processes TCP I/O and dispatches to registered handlers
- Handlers update internal lists (`m_playersInRoom`, `m_gamesInRoom`) and notify UI adapter
- Special message filtering (mute guests, ignore lists) applied in chat/private handlers

**Game lifecycle:** 
- `announceGame()` registers with metaserver
- `announcePlayersInGame()` / `announceGameStarted()` update state
- `gamesInRoomUpdate()` asynchronously pings servers for latency
- `announceGameDeleted()` unregisters

**Shutdown:** 
- Destructor calls `disconnect()` (sends logout, closes channel) and cleans up pinger

## External Dependencies
- **cseries.h**: Base types, platform macros
- **network_metaserver.h**: Class declaration, nested types
- **Message*.h** (4 files): Message framework (inflate/deflate, handlers, dispatchers)
- **preferences.h**: `network_preferences` global (mute guests, login creds)
- **alephversion.h**: Version constants, metaserver URLs, HTTPS login endpoint
- **HTTP.h**: `HTTPClient` for HTTPS key derivation
- **Logging.h**: `logAnomaly()` function
- **boost/algorithm/string/predicate.hpp**: `starts_with()` for guest name filtering
- **Standard library**: `<string>`, `<iostream>`, `<algorithm>`, `<iterator>`
- **Pinger service** (defined elsewhere): `NetRemovePinger()`, `NetCreatePinger()`, `NetGetPinger()`
