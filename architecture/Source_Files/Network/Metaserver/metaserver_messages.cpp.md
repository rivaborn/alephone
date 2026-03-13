# Source_Files/Network/Metaserver/metaserver_messages.cpp

## File Purpose
Implements TCPMess message serialization/deserialization for metaserver protocol communication. Provides encoding and decoding of client-server messages for login, game listings, player lists, and chat in the Marathon/Aleph One game.

## Core Responsibilities
- Serialize (deflate) and deserialize (inflate) metaserver protocol messages
- Manage player color, state, and auxiliary data encoding/decoding
- Parse and generate game descriptions with map/scenario metadata
- Handle room and game list message formats
- Support bidirectional chat and private message serialization
- Provide stream-based I/O helpers for padded strings and binary data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GameDescription` | struct | Game state, type, difficulty, map/scenario info, player count, time limit |
| `RoomDescription` | struct | Game room metadata: ID, player/game counts, IP address, room type |
| `MetaserverPlayerInfo` | class | Deserialized player info: ID, name, rank, colors, admin flags, away status |
| `GameListMessage::GameListEntry` | struct | Single game listing with ID, IP, port, game description, ticks |
| `HandoffToken` | typedef | 32-byte authentication token (uint8[32]) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sRoomNames[]` | `const char*[30]` | static | Map names for 30 game rooms |
| `kRoomNameCount` | const int | static | Count of room names (30) |
| `kServiceName` | `const char*` | static | "MARATHON" service identifier |
| Encryption enums | enum constants | static | kCRYPT_PLAINTEXT, kCRYPT_SIMPLE, kCRYPT_MD5, kCRYPT_HTTPS |
| State enums | enum constants | static | kSTATE_AWAKE, kSTATE_AWAY; platform detection flags |
| `kPlayerIcon`, `kAlephOneClientVersion` | uint8/uint16 constants | static | Message format constants |

## Key Functions / Methods

### write_padded_bytes
- **Signature:** `void write_padded_bytes(AOStream& inStream, const char* inBytes, size_t inByteCount, size_t inTargetLength)`
- **Purpose:** Write bytes to stream, padding with zeros to reach target length.
- **Inputs:** output stream, byte buffer, count, target length
- **Outputs/Return:** None (modifies stream)
- **Side effects:** Advances stream write position; writes null bytes
- **Calls:** `inStream.write()`, `inStream << (uint8)`

### read_string / write_string
- **Signature:** `static string read_string(AIStream& in)`, `void write_string(AOStream&, const char*)`
- **Purpose:** Read/write null-terminated C strings from/to stream.
- **Inputs:** stream, string pointer or std::string reference
- **Outputs/Return:** string (read_string)
- **Side effects:** Advances stream position; includes null terminator in write
- **Calls:** `in >> c`, `inStream.write()`

### read_padded_string
- **Signature:** `static string read_padded_string(AIStream& in, size_t length)`
- **Purpose:** Deserialize fixed-length padded (null-filled) string.
- **Inputs:** stream, length in bytes
- **Outputs/Return:** std::string (null terminator stripped)
- **Side effects:** Reads exact byte count from stream
- **Notes:** Temporary vector; zero-terminates before string conversion

### get_metaserver_player_color (overloads)
- **Signature:** `void get_metaserver_player_color(size_t colorIndex, uint16* color)` / `void get_metaserver_player_color(rgb_color color, uint16* metaserver_color)`
- **Purpose:** Convert player color to metaserver wire format (3x uint16 RGB).
- **Inputs:** color index or rgb_color struct
- **Outputs/Return:** None; writes to 3-element uint16 array
- **Side effects:** Calls `_get_player_color()` for index variant; accesses global preferences
- **Calls:** `_get_player_color()` (first overload)

### write_player_aux_data
- **Signature:** `void write_player_aux_data(AOStream& out, string name, const string& team, bool away, const string& away_message)`
- **Purpose:** Serialize player auxiliary data block (icon, state, colors, name, team).
- **Inputs:** output stream, player name, team, away flag, away message
- **Outputs/Return:** None (modifies stream)
- **Side effects:** Writes 40+ bytes; modifies name if away; accesses `network_preferences`, `player_preferences`
- **Calls:** `get_metaserver_player_color()`, `write_padded_bytes()`, `write_string()`
- **Notes:** Prepends away message prefix to name if away=true; uses custom metaserver colors if configured

### LoginAndPlayerInfoMessage::reallyDeflateTo
- **Signature:** `void reallyDeflateTo(AOStream& thePacket) const`
- **Purpose:** Serialize client login message with platform, version, credentials.
- **Inputs:** output stream
- **Outputs/Return:** None
- **Side effects:** Writes platform type, metaserver version, flags, service name, build date/time, username, player data
- **Calls:** `write_padded_string()`, `write_player_aux_data()`
- **Notes:** Detects platform at compile-time (WIN32, macOS, other); sets max_authentication to HTTPS if CURL available

### SaltMessage::reallyInflateFrom
- **Signature:** `bool reallyInflateFrom(AIStream& inStream)`
- **Purpose:** Deserialize encryption salt challenge from server.
- **Inputs:** input stream
- **Outputs/Return:** bool (true)
- **Side effects:** Populates m_encryptionType and m_salt (16 bytes)
- **Calls:** `inStream >>`, `inStream.read()`

### RoomDescription::read
- **Signature:** `void read(AIStream& inStream)`
- **Purpose:** Deserialize game room metadata.
- **Inputs:** input stream
- **Outputs/Return:** None (modifies member fields)
- **Side effects:** Parses ID, player count, IP (4 bytes), port, game count, type; ignores 10 bytes
- **Calls:** `inStream >>`, `inStream.read()`, `m_address.set_address()`, `m_address.set_port()`

### RoomListMessage::reallyInflateFrom
- **Signature:** `bool reallyInflateFrom(AIStream& inStream)`
- **Purpose:** Deserialize list of available game rooms.
- **Inputs:** input stream
- **Outputs/Return:** bool (true on success)
- **Side effects:** Populates m_rooms vector by reading until stream exhausted
- **Calls:** `RoomDescription::read()` per room

### operator>> for GameDescription
- **Signature:** `AIStream& operator>>(AIStream& stream, GameDescription& desc)`
- **Purpose:** Deserialize game description with extensible plugin-flagged fields.
- **Inputs:** input stream, GameDescription reference
- **Outputs/Return:** stream reference
- **Side effects:** Populates all GameDescription fields; conditionally reads plugin metadata based on pluginFlag bits
- **Calls:** `stream >>`, `stream.read()`, `read_padded_string()`, `stream.ignore()`
- **Notes:** Plugin flag 0x1 enables scenario metadata; 0x2 adds game options; 512-byte plugin list accommodates both

### operator<< for GameDescription
- **Signature:** `AOStream& operator<<(AOStream& stream, const GameDescription& desc)`
- **Purpose:** Serialize game description with full metadata (always writes plugin data).
- **Inputs:** output stream, GameDescription reference
- **Outputs/Return:** stream reference
- **Side effects:** Writes game type, options, time limit, checksums, difficulty, player counts, metadata, padded plugin list
- **Calls:** `write_padded_string()`, `write_padded_bytes()`
- **Notes:** Always sets pluginFlag to 0x3 (both flags enabled)

### GameListMessage::GameListEntry::game_string
- **Signature:** `string game_string() const`
- **Purpose:** Generate display string for game type (handles custom Lua scripts).
- **Inputs:** None (uses m_description member)
- **Outputs/Return:** std::string game type name
- **Side effects:** None
- **Calls:** `lua_to_game_string()`, `TS_GetCString()`
- **Notes:** Converts Lua script filename to game name; falls back to "Unknown Game Type"

### GameListMessage::GameListEntry::format_for_chat
- **Signature:** `string format_for_chat(const string& player_name) const`
- **Purpose:** Format game entry as human-readable chat message.
- **Inputs:** player name string
- **Outputs/Return:** formatted message string (e.g., "player is hosting X minutes of map_name, game_type")
- **Side effects:** None
- **Calls:** `game_string()`

### GameListMessage::reallyInflateFrom
- **Signature:** `bool reallyInflateFrom(AIStream& inStream)`
- **Purpose:** Deserialize list of available games.
- **Inputs:** input stream
- **Outputs/Return:** bool (true on success)
- **Side effects:** Populates m_entries vector with GameListEntry items and machine tick counts
- **Calls:** `operator>>` for GameListEntry, `machine_tick_count()`

### operator>> for GameListMessage::GameListEntry
- **Signature:** `AIStream& operator>>(AIStream& stream, GameListMessage::GameListEntry& entry)`
- **Purpose:** Deserialize single game list entry.
- **Inputs:** input stream, entry reference
- **Outputs/Return:** stream reference
- **Side effects:** Populates game ID, IP address (4 bytes), port, verb, enable flag, time remaining, host ID, length, and GameDescription
- **Calls:** `stream >>`, `operator>>` for GameDescription

### PrivateMessage / ChatMessage constructors and deflate/inflate
- **Signature:** `PrivateMessage(uint32 inSenderID, const string& inSenderName, uint32 inSelectedID, const string& inMessage)` / `ChatMessage(...)`
- **Purpose:** Initialize directed or broadcast chat message; serialize/deserialize message body.
- **Inputs:** sender ID, sender name, (for private: recipient ID), message text
- **Outputs/Return:** None / bool
- **Side effects:** Constructor calls `get_metaserver_player_color()`; deflate writes 3 RGB uint16s and metadata; inflate reads same
- **Calls:** `get_metaserver_player_color()`, `write_string()`, `read_string()`, `stream.write()`, `stream.read()`
- **Notes:** Private message sets kDirectedBit flag; chat message leaves flags clear; both encode sender color

### MetaserverPlayerInfo constructor
- **Signature:** `MetaserverPlayerInfo(AIStream& inStream)`
- **Purpose:** Deserialize complete player info record from metaserver.
- **Inputs:** input stream
- **Outputs/Return:** None (initializes object)
- **Side effects:** Reads verb, admin flags, player/room IDs, rank, colors, name, team, status; sets m_target to false
- **Calls:** `inStream >>`, `inStream.read()`, `inStream.ignore()`, `read_string()`
- **Notes:** Parses admin flags as bitmask (Bungie=0x1, Admin=0x4); ignores padding bytes

### PlayerListMessage::reallyInflateFrom
- **Signature:** `bool reallyInflateFrom(AIStream& inStream)`
- **Purpose:** Deserialize list of connected players.
- **Inputs:** input stream
- **Outputs/Return:** bool (true)
- **Side effects:** Populates m_players vector by constructing MetaserverPlayerInfo objects from stream
- **Calls:** MetaserverPlayerInfo constructor (loop)

### CreateGameMessage::reallyDeflateTo
- **Signature:** `void reallyDeflateTo(AOStream& thePacket) const`
- **Purpose:** Serialize game creation request.
- **Inputs:** output stream
- **Outputs/Return:** None
- **Side effects:** Writes game port, remote hub ID, and full GameDescription
- **Calls:** `operator<<` for GameDescription

### StartGameMessage::reallyDeflateTo
- **Signature:** `void reallyDeflateTo(AOStream& thePacket) const`
- **Purpose:** Serialize game start command with time.
- **Inputs:** output stream
- **Outputs/Return:** None
- **Side effects:** Writes game time (seconds) and two unused 32-bit zeros
- **Calls:** `thePacket <<`

---

## Control Flow Notes
File implements the serialization layer for metaserver protocol messages. Normal message flow:
1. Client creates typed message objects (e.g., `LoginAndPlayerInfoMessage`)
2. Framework calls `reallyDeflateTo()` to convert to wire format
3. Server receives bytes, framework calls `reallyInflateFrom()` to reconstruct objects
4. Server handler processes message and may send responses (e.g., `RoomListMessage`, `PlayerListMessage`)
5. Chat/private messages flow bidirectionally (kBOTH_* types)

No explicit initialization or shutdown visible here; message creation/destruction is caller-managed via framework (`Message::clone()`, `delete`).

## External Dependencies
**Notable includes:**
- `Message.h`, `MessageDispatcher.h`, `MessageHandler.h`, `MessageInflater.h` ΓÇö message dispatch framework
- `AStream.h` ΓÇö serialization streams (`AIStream`, `AOStream`)
- `preferences.h` ΓÇö global `network_preferences`, `player_preferences`
- `shell.h` ΓÇö `get_player_color()`
- `network_dialogs.h` ΓÇö game type constants
- `TextStrings.h` ΓÇö `TS_GetCString()` for localized strings
- `map.h` ΓÇö `TICKS_PER_SECOND` constant
- `<boost/algorithm/string/predicate.hpp>` ΓÇö `ends_with()`

**Defined elsewhere (used but not declared here):**
- `RGBColor`, `rgb_color` ΓÇö color types from cseries.h
- `IPaddress` ΓÇö network address type (supports `set_address()`, `set_port()`)
- `Scenario::instance()` ΓÇö singleton for scenario/map metadata
- `_get_player_color()` ΓÇö platform-specific color lookup
- `machine_tick_count()` ΓÇö system tick timer
- `TS_GetCString()` ΓÇö text string resource lookup
- Global preferences objects (`network_preferences`, `player_preferences`)
