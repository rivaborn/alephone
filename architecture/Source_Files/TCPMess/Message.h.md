# Source_Files/TCPMess/Message.h

## File Purpose
Defines an abstract message system for TCP communication with support for multiple serialization strategies. Provides base interfaces and concrete implementations (typed values, dataless messages, large binary chunks) for network message marshaling and unmarshaling in a game engine context.

## Core Responsibilities
- Define abstract `Message` interface with type identification, inflation (deserialization), and deflation (serialization)
- Manage raw binary data representation via `UninflatedMessage`
- Support stream-based serialization for small, typed messages via `SmallMessageHelper` and `SimpleMessage`
- Handle large binary payloads with buffer ownership management via `BigChunkOfDataMessage`
- Provide zero-payload messages via `DatalessMessage`
- Enable message cloning for copy semantics across all message types

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Message` | abstract class | Base interface for all message types; defines type ID, inflation, deflation, and cloning contracts |
| `UninflatedMessage` | class | Raw binary buffer wrapper with type ID; represents unparsed incoming/outgoing data |
| `SmallMessageHelper` | abstract class | Base for stream-based serialization; delegates to `reallyInflateFrom`/`reallyDeflateTo` |
| `BigChunkOfDataMessage` | class | Manages large binary payloads with explicit buffer ownership and copy semantics |
| `SimpleMessage<tValueType>` | template class | Typed message for single values; uses stream I/O for serialization |
| `DatalessMessage<tMessageType>` | template class | Message with type ID only, no payload data |

## Global / File-Static State
None.

## Key Functions / Methods

### Message::type()
- **Signature:** `virtual MessageTypeID type() const = 0`
- **Purpose:** Return the message's type identifier
- **Inputs:** None
- **Outputs/Return:** `MessageTypeID` (Uint16)
- **Notes:** Pure virtual; all subclasses must implement

### Message::inflateFrom()
- **Signature:** `virtual bool inflateFrom(const UninflatedMessage& inUninflated) = 0`
- **Purpose:** Parse message data from raw binary representation
- **Inputs:** `UninflatedMessage` with binary payload
- **Outputs/Return:** `bool` (may return false or raise exception on failure)
- **Notes:** Pure virtual; enables polymorphic deserialization

### Message::deflate()
- **Signature:** `virtual UninflatedMessage* deflate() const = 0`
- **Purpose:** Serialize message to raw binary form for transmission
- **Inputs:** None
- **Outputs/Return:** Heap-allocated `UninflatedMessage*` (caller owns)
- **Notes:** Pure virtual; caller responsible for cleanup

### UninflatedMessage::UninflatedMessage()
- **Signature:** `UninflatedMessage(MessageTypeID inType, size_t inLength, Uint8* inBytes = NULL)`
- **Purpose:** Construct raw message buffer; optionally take ownership of existing bytes or allocate new
- **Inputs:** `inType` (message type ID), `inLength` (buffer size), `inBytes` (optional pre-allocated buffer)
- **Outputs/Return:** Object instance
- **Side effects:** Allocates new buffer if `inBytes == NULL`
- **Notes:** Takes ownership of `inBytes` without copying if provided

### UninflatedMessage::copyToThis()
- **Signature:** `void copyToThis(const UninflatedMessage& inSource)`
- **Purpose:** Deep-copy another `UninflatedMessage`'s type and buffer
- **Inputs:** Source `UninflatedMessage`
- **Outputs/Return:** None
- **Side effects:** Allocates new buffer, deallocates old
- **Calls:** `memcpy()`

### BigChunkOfDataMessage::copyBufferFrom()
- **Signature:** `void copyBufferFrom(const Uint8* inBuffer, size_t inLength)`
- **Purpose:** Update internal buffer with new data
- **Inputs:** Pointer to buffer, length
- **Outputs/Return:** None
- **Side effects:** Deallocates old buffer, allocates and copies new

### SimpleMessage<tValueType>::reallyInflateFrom()
- **Signature:** `bool reallyInflateFrom(AIStream& inStream)`
- **Purpose:** Extract typed value from stream during deserialization
- **Inputs:** `AIStream` positioned at message data
- **Outputs/Return:** `bool` (success)
- **Side effects:** Updates `mValue` via stream extraction operator
- **Calls:** `operator>>` on stream

### SimpleMessage<tValueType>::reallyDeflateTo()
- **Signature:** `void reallyDeflateTo(AOStream& inStream) const`
- **Purpose:** Write typed value to stream during serialization
- **Inputs:** `AOStream` for output
- **Outputs/Return:** None
- **Side effects:** Stream position advances
- **Calls:** `operator<<` on stream

### DatalessMessage<tMessageType>::inflateFrom()
- **Signature:** `bool inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Validate incoming message is correct type and empty
- **Inputs:** `UninflatedMessage`
- **Outputs/Return:** `bool` (true if type matches `kType` and length is 0)
- **Notes:** No data to parse; only validates message shape

## Control Flow Notes
Part of a bidirectional message serialization pipeline:
1. **Receive path:** Binary data ΓåÆ `UninflatedMessage` ΓåÆ `inflateFrom()` ΓåÆ typed `Message`
2. **Send path:** Typed `Message` ΓåÆ `deflate()` ΓåÆ `UninflatedMessage` ΓåÆ binary for TCP
- `SmallMessageHelper` subclasses use `AIStream`/`AOStream` for structured serialization
- `BigChunkOfDataMessage` and `UninflatedMessage` handle raw buffers directly
- `DatalessMessage` optimizes for type-only messages (no payload overhead)

## External Dependencies
- `<string.h>` ΓÇö `memcpy()` for buffer copying
- `<SDL2/SDL.h>` ΓÇö `Uint16`, `Uint8` type definitions
- `AIStream`, `AOStream` (forward declared; defined elsewhere) ΓÇö Stream-based input/output for typed serialization
