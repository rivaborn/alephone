# Source_Files/TCPMess/Message.cpp

## File Purpose
Implements message serialization and deserialization for TCP-based network communication. Provides two concrete message handling strategies: `SmallMessageHelper` for structured data using stream serialization, and `BigChunkOfDataMessage` for opaque binary buffers. Both inherit from the `Message` interface and implement inflation (deserialization) and deflation (serialization) to/from `UninflatedMessage` wire format.

## Core Responsibilities
- Inflate `UninflatedMessage` objects (deserialize from wire format) into typed messages
- Deflate typed messages into `UninflatedMessage` objects (serialize to wire format)
- Manage heap-allocated buffers for `BigChunkOfDataMessage` with proper ownership semantics
- Bridge between stream-based serialization (`AIStreamBE`/`AOStreamBE`) and raw message buffers
- Support cloning and buffer copying for message replication

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SmallMessageHelper` | class | Abstract base for structured messages serialized via streams; delegates to `reallyInflateFrom`/`reallyDeflateTo` |
| `BigChunkOfDataMessage` | class | Concrete message class for opaque binary data; owns and manages a byte buffer |
| `UninflatedMessage` | class | Wire format wrapper; holds type ID and raw buffer (see Message.h) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kSmallMessageBufferSize` | enum constant (int) | file-static | Fixed 4 KiB buffer size for small message serialization |

## Key Functions / Methods

### SmallMessageHelper::inflateFrom
- **Signature:** `bool inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Deserialize structured message from wire format by wrapping buffer in a big-endian input stream.
- **Inputs:** `UninflatedMessage` containing raw bytes and length.
- **Outputs/Return:** `bool` ΓÇö success/failure (may raise exception if subclass's `reallyInflateFrom` fails).
- **Side effects:** Calls virtual `reallyInflateFrom` to populate subclass members.
- **Calls:** `AIStreamBE` constructor, `reallyInflateFrom()` (virtual, defined in subclass).
- **Notes:** Assumes buffer lifetime exceeds stream usage; stream does not own buffer.

### SmallMessageHelper::deflate
- **Signature:** `UninflatedMessage* deflate() const`
- **Purpose:** Serialize structured message to wire format using a temporary stack buffer.
- **Inputs:** None (uses `this`).
- **Outputs/Return:** Heap-allocated `UninflatedMessage*` with serialized content; caller must `delete`.
- **Side effects:** Allocates 4 KiB temporary vector; calls `reallyDeflateTo` to serialize; allocates and copies into result.
- **Calls:** `std::vector` allocation, `AOStreamBE` constructor, `reallyDeflateTo()` (virtual), `memcpy`, `new`.
- **Notes:** Fixed 4 KiB buffer may truncate oversized serialized data; no overflow check.

### BigChunkOfDataMessage::BigChunkOfDataMessage (constructor)
- **Signature:** `BigChunkOfDataMessage(MessageTypeID inType, const byte* inBuffer, size_t inLength)`
- **Purpose:** Construct a message containing raw binary data.
- **Inputs:** Message type ID; optional buffer pointer and length.
- **Outputs/Return:** Initialized object; buffer owned by this instance.
- **Side effects:** Calls `copyBufferFrom` to allocate and copy buffer.
- **Calls:** `copyBufferFrom()`.
- **Notes:** Default constructor arguments allow empty buffer initialization.

### BigChunkOfDataMessage::inflateFrom
- **Signature:** `bool inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Deserialize by copying buffer contents from `UninflatedMessage`.
- **Inputs:** `UninflatedMessage` with raw data.
- **Outputs/Return:** Always returns `true`.
- **Side effects:** Calls `copyBufferFrom`, which deletes old buffer and allocates new one.
- **Calls:** `copyBufferFrom()`.
- **Notes:** No validation of message type; simply overwrites existing buffer.

### BigChunkOfDataMessage::deflate
- **Signature:** `UninflatedMessage* deflate() const`
- **Purpose:** Serialize by creating `UninflatedMessage` with copy of buffer.
- **Inputs:** None (uses `this`).
- **Outputs/Return:** Heap-allocated `UninflatedMessage*`; caller must `delete`.
- **Side effects:** Allocates new `UninflatedMessage`; copies buffer.
- **Calls:** `new UninflatedMessage`, `memcpy`.
- **Notes:** Zero-copy would require buffer ownership transfer (not implemented).

### BigChunkOfDataMessage::copyBufferFrom
- **Signature:** `void copyBufferFrom(const byte* inBuffer, size_t inLength)`
- **Purpose:** Replace internal buffer with a copy of provided data.
- **Inputs:** Buffer pointer and length; may be `NULL` if `inLength == 0`.
- **Outputs/Return:** None.
- **Side effects:** Deletes existing buffer; allocates and copies new buffer.
- **Calls:** `delete[]`, `new[]`, `memcpy`.
- **Notes:** Handles empty buffer case (`mBuffer = NULL`); assumes `inLength` is accurate.

### BigChunkOfDataMessage::clone
- **Signature:** `BigChunkOfDataMessage* clone() const`
- **Purpose:** Create a deep copy of this message.
- **Inputs:** None (uses `this`).
- **Outputs/Return:** Heap-allocated `BigChunkOfDataMessage*`; caller must `delete`.
- **Side effects:** Allocates new object and buffer.
- **Calls:** Constructor, `copyBufferFrom`.
- **Notes:** Fully independent copy; safe to modify or delete either instance.

### BigChunkOfDataMessage::~BigChunkOfDataMessage (destructor)
- **Signature:** `~BigChunkOfDataMessage()`
- **Purpose:** Free allocated buffer on object destruction.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Deallocates `mBuffer` via `delete[]`.
- **Calls:** `delete[]`.
- **Notes:** Safe to call on object with `mBuffer == NULL` (empty message).

## Control Flow Notes
This file is part of a **message serialization layer** for TCP networking. Likely usage:
- **Inflate (receive path):** Wire bytes ΓåÆ `UninflatedMessage` ΓåÆ call `Message::inflateFrom()` to populate application-level message.
- **Deflate (send path):** Application message ΓåÆ call `Message::deflate()` ΓåÆ `UninflatedMessage` ΓåÆ wire transmission.

`SmallMessageHelper` and `BigChunkOfDataMessage` are two strategies:
- Structured, variable-sized messages (e.g., game events) use `SmallMessageHelper` with stream operators.
- Opaque bulk data (e.g., map files, replays) use `BigChunkOfDataMessage` with raw buffer copy.

Networking is only active when `DISABLE_NETWORKING` is not defined (conditional compilation guard).

## External Dependencies
- **`Message.h`** ΓÇö base `Message` interface, `UninflatedMessage` class, abstract `SmallMessageHelper`, `BigChunkOfDataMessage` declarations.
- **`<string.h>`** ΓÇö `memcpy()` for buffer operations.
- **`<vector>`** ΓÇö `std::vector<byte>` temporary buffer in `SmallMessageHelper::deflate()`.
- **`AStream.h`** ΓÇö `AIStreamBE` (big-endian input stream), `AOStreamBE` (big-endian output stream) for structured serialization.
- **Defined elsewhere:** `AIStreamBE`, `AOStreamBE`, `UninflatedMessage`, stream operator `>>` and `<<` overloads.
