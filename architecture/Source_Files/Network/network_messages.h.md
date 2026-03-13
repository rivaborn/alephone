# Source_Files/Network/network_messages.h

## File Purpose
Defines message type constants and classes for network communication in Aleph One's game setup and multiplayer protocol. Implements a type-safe message framework for negotiating game join, capabilities, topology, large data distribution (map/physics/Lua), chat, and remote hub commands over TCPMess.

## Core Responsibilities
- Enumerate message type IDs (kHELLO_MESSAGE through kREMOTE_HUB_REQUEST_MESSAGE)
- Provide template-based message classes for type-safe, compile-time message ID binding
- Define concrete message classes for join handshake, capabilities negotiation, player acceptance
- Support large data distribution with optional compression (zipped variants)
- Implement serialization/deserialization via AStream deflate/inflate pattern
- Manage client connection state and message dispatch handlers

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| kHELLO_MESSAGE, kJOINER_INFO_MESSAGE, kACCEPT_JOIN_MESSAGE, etc. | enum | Message type constants (700ΓÇô724) for identifying message kinds |
| TemplatizedSimpleMessage | template class | Binds a compile-time message type ID to SimpleMessage<tValueType> |
| HelloMessage | class (SmallMessageHelper) | Version handshake for protocol negotiation |
| JoinerInfoMessage | class (SmallMessageHelper) | Prospective joiner metadata and version |
| AcceptJoinMessage | class (SmallMessageHelper) | Accept/reject join response with NetPlayer data |
| CapabilitiesMessage | class (SmallMessageHelper) | Capabilities map for feature negotiation |
| TopologyMessage | class (SmallMessageHelper) | Network topology (players, identifiers, game/server data) |
| TemplatizedDataMessage | template class | Wraps large data buffers with compile-time message type ID |
| MapMessage, ZippedMapMessage, PhysicsMessage, ZippedPhysicsMessage, LuaMessage, ZippedLuaMessage | typedef | Large chunk messages for scenario, physics, script distribution |
| NetworkChatMessage | class (SmallMessageHelper) | Targeted chat (to players, teams, or clients) up to 1024 bytes |
| ServerWarningMessage | class (SmallMessageHelper) | Server warnings with reason codes (e.g., kJoinerUngatherable) |
| ClientInfoMessage | class (SmallMessageHelper) | Chat participant metadata (add/update/remove) with action enum |
| RemoteHubCommandMessage | class (SmallMessageHelper) | Remote hub protocol command with optional int data |
| RemoteHubHostConnectMessage | class (HelloMessage subclass) | Connection request to remote hub (inherits version field) |
| RemoteHubHostResponseMessage | class (SmallMessageHelper) | Remote hub acceptance/rejection response |
| RemoteHubReadyMessage | class (SmallMessageHelper) | Remote hub ready signal (zero-byte payload) |
| Client | struct | Manages single client connection state, message handlers, and dispatcher |

## Global / File-Static State
None.

## Key Functions / Methods

### TemplatizedSimpleMessage
- **Signature:** template-based constructor and `clone()`, `type()`
- **Purpose:** Provides message type binding at compile-time; avoids runtime type lookups
- **Inputs:** Optional tValueType value in constructor
- **Outputs/Return:** `clone()` returns new instance; `type()` returns kType
- **Side effects:** None (data holder)
- **Calls:** SimpleMessage<tValueType> constructor
- **Notes:** Base pattern for JoinPlayerMessage typedef; ensures type safety

### HelloMessage / RemoteHubHostConnectMessage
- **Signature:** `HelloMessage(const std::string& version)`, `version()` getter/setter
- **Purpose:** Version negotiation during initial handshake; RemoteHubHostConnectMessage reuses for remote hub connection
- **Inputs:** Version string (e.g., "01.00")
- **Outputs/Return:** `version()` returns mVersion string
- **Side effects:** `reallyDeflateTo/reallyInflateFrom` serialize/deserialize version via streams
- **Calls:** SmallMessageHelper base serialization
- **Notes:** RemoteHubHostConnectMessage inherits directly; no new fields

### AcceptJoinMessage
- **Signature:** `AcceptJoinMessage(bool accepted, NetPlayer* player)`
- **Purpose:** Server's response to joiner request; contains acceptance flag and player data
- **Inputs:** bool accepted, NetPlayer pointer
- **Outputs/Return:** `accepted()` getter/setter, `player()` returns &mPlayer
- **Side effects:** Copies player data on construction; serialization via AStream
- **Calls:** SmallMessageHelper deflate/inflate
- **Notes:** Player data copied, not referenced; mPlayer is by-value member

### CapabilitiesMessage / TopologyMessage
- **Signature:** `CapabilitiesMessage(const Capabilities& caps)`, `TopologyMessage(NetTopology* topo)`
- **Purpose:** Negotiate protocol features (map version, Lua, Rugby, zipped data); distribute network topology (players, identifiers)
- **Inputs:** Capabilities map or NetTopology struct
- **Outputs/Return:** `capabilities()`, `topology()` return pointers to member objects
- **Side effects:** Copy on construction; serialization stores/retrieves via streams
- **Calls:** SmallMessageHelper serialization
- **Notes:** mTopology and mCapabilities are by-value members; pointers return &member

### NetworkChatMessage
- **Signature:** `NetworkChatMessage(const char* text, int16 senderID, int16 target, int16 targetID)`
- **Purpose:** Deliver chat message with routing metadata (to specific player, team, or all clients)
- **Inputs:** Chat text (max 1024), sender ID, target type enum (kTargetPlayers, kTargetTeam, kTargetClient, etc.), target ID
- **Outputs/Return:** Accessors for chatText, senderID, target, targetID
- **Side effects:** strncpy with null termination; serialization
- **Calls:** SmallMessageHelper deflate/inflate
- **Notes:** Fixed-size buffer (CHAT_MESSAGE_SIZE); null-termination enforced

### BigChunkOfZippedDataMessage
- **Signature:** Constructor delegating to BigChunkOfDataMessage; `inflateFrom()`, `deflate()`
- **Purpose:** Transparent gzip compression/decompression of large buffers on serialize/deserialize
- **Inputs:** MessageTypeID, uint8* buffer, size_t length
- **Outputs/Return:** `deflate()` returns zipped UninflatedMessage; `inflateFrom()` unzips on deserialize
- **Side effects:** Allocates compressed/uncompressed buffers; I/O compression
- **Calls:** zlib (via base class or implementation elsewhere)
- **Notes:** Inherits buffer management from BigChunkOfDataMessage; zip/unzip is transparent to caller

### Client::handleX (message handlers)
- **Signature:** `void handleJoinerInfoMessage(JoinerInfoMessage*, CommunicationsChannel*)`, etc.
- **Purpose:** Dispatch incoming messages to state-specific logic (joiner info, capabilities, join accept, chat, remote hub commands, color change)
- **Inputs:** Typed message pointer, communications channel
- **Outputs/Return:** void
- **Side effects:** Update client state, send responses via channel, call mDispatcher or handler objects
- **Calls:** MessageDispatcher and MessageHandler objects (mDispatcher, mJoinerInfoMessageHandler, etc.)
- **Notes:** Multiple handler methods; mDispatcher likely delegates to mXxxMessageHandler members based on message type

### Client::drop()
- **Signature:** `void drop()`
- **Purpose:** Disconnect and clean up client connection
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Closes channel; updates state to _disconnect
- **Calls:** Likely CommunicationsChannel::close() or similar

### Client::capabilities_indicate_player_is_gatherable() / can_pregame_chat()
- **Signature:** `bool capabilities_indicate_player_is_gatherable(bool warn_joiner)`, `bool can_pregame_chat()`
- **Purpose:** Predicate checks for client capability and state
- **Inputs:** warn_joiner flag (optional); no inputs for can_pregame_chat
- **Outputs/Return:** bool (gatherable, can chat)
- **Side effects:** warn_joiner may send warning message
- **Calls:** Possibly ServerWarningMessage construction
- **Notes:** can_pregame_chat checks state membership against allowed states (_connected, _connected_but_not_yet_shown, _ungatherable, _joiner_didnt_accept, _awaiting_accept_join, _awaiting_map)

## Control Flow Notes

**Init/Join Phase:**
1. Joiner sends HelloMessage with version ΓåÆ Gatherer validates and responds
2. Joiner sends JoinerInfoMessage (prospective_joiner_info, version) ΓåÆ Gatherer evaluates
3. Gatherer sends CapabilitiesMessage ΓåÆ Joiner checks compatibility
4. Joiner sends CapabilitiesMessage back ΓåÆ Gatherer validates
5. Gatherer sends AcceptJoinMessage ΓåÆ Joiner accepted or rejected
6. If accepted, Gatherer sends TopologyMessage (players, game state, server addr)
7. Gatherer streams large data: MapMessage, PhysicsMessage, LuaMessage (or zipped variants)
8. Joiner sends AcceptJoinMessage back (if required) ΓåÆ game starts

**Pregame Phase:**
- NetworkChatMessage for chat (client can send if in allowed states)
- ClientInfoMessage to track chat participants (add/update/remove)
- ServerWarningMessage for server alerts (e.g., ungatherable joiner)

**Remote Hub Extension:**
- RemoteHubHostConnectMessage (HelloMessage variant) initiates remote hub request
- RemoteHubCommandMessage carries hub commands (accept/end game) with data
- RemoteHubReadyMessage signals hub is ready
- RemoteHubHostResponseMessage returns hub's accept/reject

**In-Game:**
- NetworkStatsMessage reports latency/jitter/errors

## External Dependencies
- **cseries.h**: Core platform/type definitions (int16, uint32, assert)
- **AStream.h**: AIStream, AOStream classes for endian-aware serialization (operator>>, operator<<)
- **Message.h**: SmallMessageHelper (deflate/inflate), BigChunkOfDataMessage, SimpleMessage<T>, DatalessMessage<tMessageType>, UninflatedMessage
- **network_capabilities.h**: Capabilities class (map of stringΓåÆuint32 feature versions)
- **network_private.h**: NetPlayer, NetTopology, CommunicationsChannel, MessageDispatcher, MessageHandler, ClientChatInfo, RemoteHubCommand enum, prospective_joiner_info struct
- **Standard library**: `<string>`, `<memory>` (std::string, std::shared_ptr, std::unique_ptr, std::vector)
- **SDL2**: Uint8, Uint16 types from SDL.h
