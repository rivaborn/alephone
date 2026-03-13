# Subsystem Overview

## Purpose
TCPMess provides reliable, message-based TCP networking for peer-to-peer multiplayer communication. It manages bidirectional socket connections, asynchronous/synchronous message queuing and dispatch, serialization/deserialization with optional compression, and handler-based message routing by type.

## Key Files
| File | Role |
|------|------|
| CommunicationsChannel.h | TCP connection lifecycle, message queue management, async pump and blocking receive APIs |
| CommunicationsChannel.cpp | Socket I/O, message framing (8-byte header + body), timeout tracking, batch flushing |
| Message.h | Abstract message interface; defines `Message`, `UninflatedMessage`, `SmallMessageHelper`, `BigChunkOfDataMessage`, `DatalessMessage` |
| Message.cpp | Serialization/deserialization; bridges stream-based (`AIStreamBE`/`AOStreamBE`) and raw buffer formats |
| MessageHandler.h | Abstract handler interface; template wrappers for function pointers and member methods |
| MessageDispatcher.h | Type-ID message router; maps message type IDs to handler objects with fallback support |
| MessageInflater.h | Prototype-pattern registry for message deserialization; factory for instantiating typed messages |
| MessageInflater.cpp | Inflation implementation; clones prototypes and deserializes `UninflatedMessage` objects |

## Core Responsibilities
- Establish, maintain, and tear down TCP socket connections (client-initiated or server-accepted)
- Queue and transmit outgoing messages with framing; batch-flush multiple channels
- Receive, buffer, and parse incoming messages with header validation
- Serialize/deserialize messages with optional inflation (decompression) support
- Route messages to registered handlers based on message type ID
- Provide synchronous blocking receive with overall and inactivity timeout enforcement
- Track connection activity timestamps for timeout and heartbeat monitoring
- Support message type filtering and cloning for safe message replication

## Key Interfaces & Data Flow
**Exposes:**
- `CommunicationsChannel` ΓÇö constructor, `connect()`, `enqueueOutgoingMessage()`, `flushOutgoingMessages()`, `pump()`, blocking receive methods
- `Message` base class ΓÇö `type()`, `inflate()`, `deflate()`, `clone()`
- `MessageHandler` interface ΓÇö polymorphic `handle()` callback
- `MessageDispatcher` ΓÇö routes `Message` to registered handlers by type ID
- `MessageInflater` ΓÇö registry of message prototypes for deserialization

**Consumes:**
- `NetworkInterface` ΓÇö `TCPsocket`, `TCPlistener`, `IPaddress` for socket creation
- `AStream` (AIStreamBE, AOStreamBE) ΓÇö stream-based serialization for typed messages
- `machine_tick_count()`, `sleep_for_machine_ticks()` from cseries ΓÇö timing and blocking

## Runtime Role
- **Init:** Register message prototypes via `MessageInflater`; create `CommunicationsChannel` instances or accept connections
- **Frame:** Call `pump()` on channels to process queued incoming/outgoing messages; route dispatched messages to handlers
- **Shutdown:** Flush pending messages, close TCP sockets, deallocate queued messages and prototypes

## Notable Implementation Details
- Uses **prototype pattern** for message inflation: registered prototype messages are cloned and populated from wire data
- **Header + body framing:** 8-byte header (likely type ID + size) followed by message body
- **Dual I/O modes:** asynchronous pump-based processing (non-blocking) and synchronous blocking receive with cumulative/inactivity timeout enforcement
- **Message ownership:** `BigChunkOfDataMessage` manages heap-allocated buffers; `UninflatedMessage` is wire representation
- **Conditional compilation:** `#if !defined(DISABLE_NETWORKING)` allows build-time networking disable
- **Memento pattern:** `CommunicationsChannel` stores client-defined context for safe downcasting
