# Source_Files/Network/network_messages.cpp

## File Purpose
Implements serialization and deserialization of network protocol messages for Aleph One multiplayer game setup and communication. Handles conversion of message objects to/from byte streams, including compression of large data chunks using zlib.

## Core Responsibilities
- Serialize message objects to byte streams via `reallyDeflateTo()` methods
- Deserialize byte streams back to message objects via `reallyInflateFrom()` methods
- Compress/decompress large message payloads (maps, physics, Lua scripts)
- Serialize/deserialize player network information (addresses, ports, player data)
- Support variable-length network data (capabilities, chat messages, topology info)
- Provide utility functions for string and structured data I/O across stream types

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| NetPlayer | struct | Player network identity: addresses, stream ID, player metadata |
| NetTopology | struct | Game state snapshot: player list, game settings, server address |
| NetServer | struct | Server network address pair (DSP and DDP) |
| BigChunkOfZippedDataMessage | class | Container for compressed map/physics/Lua payloads |
| UninflatedMessage | class | Raw message bytes (defined elsewhere) |
| NetworkStats | struct | Per-player latency, jitter, error counts, pregame state |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| write_string | function | static | Helper to serialize C-strings with null terminator |
| read_string | function | static | Helper to deserialize C-strings |
| deflateNetPlayer | function | static | Serialize NetPlayer struct fields |
| inflateNetPlayer | function | static | Deserialize NetPlayer struct fields |

## Key Functions / Methods

### write_string
- **Signature:** `static void write_string(AOStream& outputStream, const char *s)`
- **Purpose:** Write a null-terminated C-string to an output stream.
- **Inputs:** Output stream, C-string pointer
- **Outputs/Return:** None; writes to stream
- **Side effects:** Advances stream position by `strlen(s) + 1` bytes
- **Calls:** `AOStream::write()`
- **Notes:** Includes null terminator in the stream.

### read_string
- **Signature:** `static void read_string(AIStream& inputStream, char *s, size_t length)`
- **Purpose:** Read a null-terminated C-string from input stream with size bounds.
- **Inputs:** Input stream, output buffer, max length
- **Outputs/Return:** Writes to buffer `s`; adds null terminator
- **Side effects:** Advances stream position; stops early if length exceeded
- **Calls:** `AIStream::operator>>(int8&)`
- **Notes:** Reads byte-by-byte until `\0` or buffer full; always null-terminates.

### BigChunkOfZippedDataMessage::inflateFrom
- **Signature:** `bool inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Decompress a zlib-compressed message payload and store uncompressed data.
- **Inputs:** UninflatedMessage (compressed bytes with 4-byte length prefix)
- **Outputs/Return:** `true` if decompression succeeded; `false` on error
- **Side effects:** Calls `copyBufferFrom()` with uncompressed data; logs warning on failure
- **Calls:** `uncompress()` (zlib), `copyBufferFrom()`, `logWarning()`
- **Notes:** First 4 bytes are big-endian uncompressed size; returns `true` for zero-size payload.

### BigChunkOfZippedDataMessage::deflate
- **Signature:** `UninflatedMessage* deflate() const`
- **Purpose:** Compress this message's buffer using zlib, return new UninflatedMessage.
- **Inputs:** None (uses internal buffer from base class)
- **Outputs/Return:** Dynamically allocated `UninflatedMessage*` with compressed data; `nullptr` on error
- **Side effects:** Allocates new message; may log failure implicitly
- **Calls:** `compress()` (zlib), `new UninflatedMessage()`, `memcpy()`
- **Notes:** Allocates temp buffer sized at 105% + 12 bytes; returns `nullptr` on compress failure.

### AcceptJoinMessage::reallyDeflateTo / reallyInflateFrom
- **Signature (deflate):** `void reallyDeflateTo(AOStream& outputStream) const`
- **Purpose:** Serialize acceptance status and player info.
- **Inputs (deflate):** Output stream
- **Outputs/Return:** Bytes written to stream
- **Calls (deflate):** `deflateNetPlayer()`, stream `operator<<`
- **Notes:** Writes boolean accepted flag then player data.

### TopologyMessage::reallyDeflateTo / reallyInflateFrom
- **Signature (deflate):** `void reallyDeflateTo(AOStream& outputStream) const`
- **Purpose:** Serialize complete game topology (all players, server, game settings).
- **Inputs (deflate):** Output stream
- **Outputs/Return:** All topology fields serialized to stream
- **Calls (deflate):** `deflateNetPlayer()` (looped), `write_string()`, stream operators
- **Notes:** Serializes topology tag, player count, game data, all player records, and server address.

### NetworkStatsMessage::reallyDeflateTo / reallyInflateFrom
- **Signature (deflate):** `void reallyDeflateTo(AOStream& outputStream) const`
- **Purpose:** Serialize per-player network statistics (latency, jitter, error counts, pregame state).
- **Inputs (deflate):** Output stream
- **Outputs/Return:** Stat records serialized to stream
- **Calls (deflate):** Stream `operator<<` in loop over `mStats` vector
- **Notes:** Loops through all stats; inflateFrom reads until stream exhausted.

**Other Message Types:** `CapabilitiesMessage`, `ChangeColorsMessage`, `ClientInfoMessage`, `HelloMessage`, `JoinerInfoMessage`, `NetworkChatMessage`, `ServerWarningMessage`, `RemoteHubCommandMessage`, `RemoteHubHostResponseMessage` all follow the pattern of implementing `reallyDeflateTo()` and `reallyInflateFrom()` to serialize/deserialize their member fields. See header for details.

## Control Flow Notes
Messages follow a two-phase I/O pattern (likely called from base class `SmallMessageHelper`):
1. **Outbound:** Object ΓåÆ `reallyDeflateTo()` ΓåÆ stream ΓåÆ network
2. **Inbound:** Stream ΓåÆ `reallyInflateFrom()` ΓåÆ object ΓåÆ application

For large data messages (`BigChunkOfZippedDataMessage`), compression/decompression happens at the `deflate()` / `inflateFrom()` layer before general serialization.

String and structured data use helper functions (`write_string`, `read_string`, `deflateNetPlayer`, `inflateNetPlayer`) to ensure consistent encoding across multiple message types.

## External Dependencies
- **zlib.h** ΓÇö `compress()`, `uncompress()` for payload compression
- **AStream.h** ΓÇö `AIStream`, `AIStreamBE`, `AOStream`, `AOStreamBE` for typed I/O
- **network_messages.h** ΓÇö Message class definitions and enum constants
- **network_private.h** ΓÇö `NetPlayer`, `NetTopology`, `NetServer`, `ClientChatInfo`, `NetworkStats`, enum tags
- **Logging.h** ΓÇö `logWarning()` macro for error reporting
- **cseries.h** ΓÇö Platform types and utilities
