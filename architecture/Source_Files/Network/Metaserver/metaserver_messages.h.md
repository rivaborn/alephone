# Source_Files/Network/Metaserver/metaserver_messages.h

## File Purpose
Defines TCPMess message types and serializable message classes for metaserver clientΓÇôserver communication. Covers login/authentication, game/player/room listings, chat, private messaging, and remote hub discovery in the Aleph One game engine.

## Core Responsibilities
- Enumerate message type constants (kSERVER_*, kCLIENT_*, kBOTH_*)
- Define serializable message classes inheriting from `SmallMessageHelper` or template `DatalessMessage`
- Implement serialization/deserialization via `reallyInflateFrom()` and `reallyDeflateTo()` methods
- Encapsulate game metadata, player profiles, room/server descriptions, and chat/private message content
- Support authentication flow (salt exchange, handoff tokens)
- Enable game listing, player roster updates, and dynamic room/remote-hub discovery

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `HandoffToken` | typedef | 32-byte authentication token for room login |
| `GameDescription` | struct | Game metadata: type, time/kill limits, map checksum, difficulty, player count, scenario/physics/net protocol IDs, latency |
| `MetaserverPlayerInfo` | class | Player profile: ID, name, admin flags, rank, status, colors, away status |
| `GameListMessage::GameListEntry` | nested struct | Game list item with ID, IP/port, description, time remaining calculation, compatibility check |
| `RoomDescription` | class | Room metadata: ID, player/game counts, type (normal/ranked/tournament), server address |
| `RemoteHubServerDescription` | class | Remote hub: ID and network address |

## Global / File-Static State
None.

## Key Functions / Methods

### `GameDescription::GameDescription()` (constructor)
- **Signature:** `GameDescription()`
- **Purpose:** Initialize a game description with default values
- **Inputs:** None
- **Outputs/Return:** New instance
- **Side effects:** Calls `Scenario::instance()->GetID()`, `GetName()`, `GetVersion()` at construction time
- **Calls:** `Scenario::instance()` (singleton access)
- **Notes:** Sets default port to 8, game name to "Untitled Game", latency to `UINT16_MAX`

### `RoomDescription::read()`
- **Signature:** `void read(AIStream& inStream)`
- **Purpose:** Deserialize room info from network stream
- **Inputs:** `AIStream& inStream` ΓÇô input serialization stream
- **Outputs/Return:** None (mutates member fields)
- **Side effects:** Updates `m_id`, `m_playerCount`, `m_gameCount`, `m_type`, `m_address`
- **Calls:** `inStream >>` operators
- **Notes:** Reads 4-byte IP address and 16-bit port; address constructed via `set_address()`, `set_port()`

### `RemoteHubServerDescription::read()`
- **Signature:** `void read(AIStream& inStream)`
- **Purpose:** Deserialize remote hub server info
- **Inputs:** `AIStream& inStream`
- **Outputs/Return:** None
- **Side effects:** Updates `m_id`, `m_address` (IP and port)
- **Calls:** `inStream >>` operators, `m_address.set_address()`, `m_address.set_port()`

### `MetaserverPlayerInfo::sort()` (static comparator)
- **Signature:** `static bool sort(const MetaserverPlayerInfo& a, const MetaserverPlayerInfo& b)`
- **Purpose:** Provide sort order for player lists (admin flags desc, status asc, ID asc)
- **Inputs:** Two `MetaserverPlayerInfo` references
- **Outputs/Return:** `bool` ΓÇô true if `a` should precede `b`
- **Side effects:** None
- **Calls:** None (uses member comparisons)
- **Notes:** Orders admins first, then by status, then by ID

### `GameListMessage::GameListEntry::minutes_remaining()`
- **Signature:** `int minutes_remaining() const`
- **Purpose:** Calculate time remaining in game accounting for elapsed time since last update
- **Inputs:** None (uses member fields `m_timeRemaining`, `m_ticks`)
- **Outputs/Return:** `int` ΓÇô minutes remaining (clamped to 0 if expired, or ΓÇô1 if unlimited)
- **Side effects:** Calls `machine_tick_count()` to get current SDL ticks
- **Calls:** `machine_tick_count()` (external, defined elsewhere)
- **Notes:** Subtracts elapsed milliseconds from cached time remaining

### Per-message class: `type()`, `clone()`
- **Signature:** `MessageTypeID type() const` / `ClassName* clone() const`
- **Purpose:** Return message type constant and create deep copy
- **Side effects:** `clone()` allocates via `new`
- **Notes:** All message subclasses implement these; minimal overhead

### Per-message class: `reallyInflateFrom()`, `reallyDeflateTo()`
- **Signature:** `bool reallyInflateFrom(AIStream&)` / `void reallyDeflateTo(AOStream&) const`
- **Purpose:** Deserialize / serialize message content
- **Inputs/Outputs:** Stream operators (`>>`, `<<`)
- **Notes:** Many serverΓåÆclient-only messages have stub `reallyInflateFrom()` returning `false` (not intended for receiving)

## Control Flow Notes

**Message flow direction:**
- **ServerΓåÆClient:** `kSERVER_*` messages (RoomList, PlayerList, GameList, Salt, LoginSuccess, Denial, Broadcast, etc.)
- **ClientΓåÆServer:** `kCLIENT_*` messages (Login, RoomLogin, CreateGame, StartGame, Localize, etc.)
- **Bidirectional:** `kBOTH_*` messages (Chat, PrivateMessage, KeepAlive)

**Typical auth sequence:**
1. Client receives `SaltMessage`
2. Client sends `LoginAndPlayerInfoMessage`
3. Server responds `LoginSuccessfulMessage` (with handoff token) or `DenialMessage`

**Typical gameplay setup:**
1. Client requests game/player/room lists via various `kCLIENT_*` messages
2. Server pushes updates via `RoomListMessage`, `PlayerListMessage`, `GameListMessage`
3. Client creates game via `CreateGameMessage`, or joins via `RoomLoginMessage` with token

Not bound to a specific init/frame/render loop; this is a protocol definition layer.

## External Dependencies

- **Message.h:** `SmallMessageHelper`, `DatalessMessage<>`, `Message` base class
- **AStream.h:** `AIStream`, `AOStream` (serialization stream classes, including endian variants)
- **Scenario.h:** `Scenario::instance()` (singleton for map/scenario metadata)
- **network.h:** `kNetworkSetupProtocolID` constant, `IPaddress` type
- **std::string, std::vector:** Standard library containers
- **boost::algorithm::to_lower_copy():** Case conversion utility
- **machine_tick_count():** External function (SDL ticks); defined elsewhere
- **uint8, uint16, uint32, int16, int32:** Typedef'd integer types (cstypes.h)
