# Source_Files/TCPMess/CommunicationsChannel.cpp
## File Purpose
Implements reliable message-based TCP networking with asynchronous (pump-based) and synchronous blocking receive operations. Manages bidirectional message queues, supports optional message inflation/deflation and handler callbacks, and provides a factory for accepting incoming connections.

## Core Responsibilities
- Establish, maintain, and tear down TCP socket connections
- Serialize/deserialize messages with framing (8-byte header + body)
- Manage outgoing and incoming message queues with pump-based I/O
- Provide synchronous blocking receive with overall and inactivity timeouts
- Track connection activity timestamps for timeout calculation
- Support pluggable message inflation (decompression) and message handling callbacks
- Accept incoming connections via factory pattern
- Batch-flush multiple channels efficiently

## External Dependencies
- **"CommunicationsChannel.h"** ΓÇö class declarations and type definitions
- **"AStream.h"** ΓÇö AIStreamBE (big-endian input), AOStreamBE (big-endian output) for serializing header
- **"MessageInflater.h"** ΓÇö `MessageInflater::inflate()` for message decompression
- **"MessageHandler.h"** ΓÇö `MessageHandler` interface for receive callbacks
- **"network.h"** ΓÇö `NetGetNetworkInterface()` to obtain TCP socket factory
- **"cseries.h"** ΓÇö common engine types (`Uint8`, `Uint16`, `Uint32`), `machine_tick_count()`, `sleep_for_machine_ticks()`
- **<stdlib.h>, <iostream>, <cerrno>, <algorithm>** ΓÇö standard library
- **Defined elsewhere:** `TCPsocket`, `TCPlistener`, `IPaddress` (from NetworkInterface.h), `Message`, `UninflatedMessage`, `MessageTypeID` (from Message.h), `Memento` (forward decl.), `MessageInflater`, `MessageHandler`

# Source_Files/TCPMess/CommunicationsChannel.h
## File Purpose
Implements TCP-based bidirectional message communication channels for network games. Manages socket lifecycle, message queuing (incoming/outgoing), synchronous blocking receive with timeouts, and asynchronous message dispatch. Designed for both client-initiated and server-accepted connections.

## Core Responsibilities
- Establish and manage TCP socket connections (via `connect()` or factory-created sockets)
- Queue and transmit outgoing messages via `enqueueOutgoingMessage()` and `flushOutgoingMessages()`
- Receive and buffer incoming messages; parse headers and message bodies
- Dispatch received messages to registered handler callbacks
- Provide synchronous blocking receive with overall and inactivity timeouts
- Track connection activity timestamps for timeout/heartbeat monitoring
- Store client-defined context via Memento pattern (safe downcasting)
- Support message type filtering in synchronous receive (receive only specific types)

## External Dependencies
- **STL headers:** `<list>`, `<string>`, `<memory>`, `<stdexcept>`, `<vector>`
- **NetworkInterface.h:** `TCPsocket`, `TCPlistener`, `IPaddress`
- **Message.h:** `Message`, `UninflatedMessage`, `MessageTypeID`
- **csmisc.h:** `machine_tick_count()` for timing
- **Forward declarations:** `MessageInflater` (inflates raw bytes to `Message`); `MessageHandler` (callback interface for dispatched messages)

# Source_Files/TCPMess/Message.cpp
## File Purpose
Implements message serialization and deserialization for TCP-based network communication. Provides two concrete message handling strategies: `SmallMessageHelper` for structured data using stream serialization, and `BigChunkOfDataMessage` for opaque binary buffers. Both inherit from the `Message` interface and implement inflation (deserialization) and deflation (serialization) to/from `UninflatedMessage` wire format.

## Core Responsibilities
- Inflate `UninflatedMessage` objects (deserialize from wire format) into typed messages
- Deflate typed messages into `UninflatedMessage` objects (serialize to wire format)
- Manage heap-allocated buffers for `BigChunkOfDataMessage` with proper ownership semantics
- Bridge between stream-based serialization (`AIStreamBE`/`AOStreamBE`) and raw message buffers
- Support cloning and buffer copying for message replication

## External Dependencies
- **`Message.h`** ΓÇö base `Message` interface, `UninflatedMessage` class, abstract `SmallMessageHelper`, `BigChunkOfDataMessage` declarations.
- **`<string.h>`** ΓÇö `memcpy()` for buffer operations.
- **`<vector>`** ΓÇö `std::vector<byte>` temporary buffer in `SmallMessageHelper::deflate()`.
- **`AStream.h`** ΓÇö `AIStreamBE` (big-endian input stream), `AOStreamBE` (big-endian output stream) for structured serialization.
- **Defined elsewhere:** `AIStreamBE`, `AOStreamBE`, `UninflatedMessage`, stream operator `>>` and `<<` overloads.

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

## External Dependencies
- `<string.h>` ΓÇö `memcpy()` for buffer copying
- `<SDL2/SDL.h>` ΓÇö `Uint16`, `Uint8` type definitions
- `AIStream`, `AOStream` (forward declared; defined elsewhere) ΓÇö Stream-based input/output for typed serialization

# Source_Files/TCPMess/MessageDispatcher.h
## File Purpose
Implements a message-routing dispatcher that maps message type IDs to handler objects. Routes incoming messages to the appropriate handler based on type, with support for a default fallback handler. Acts as the central message router in a network communication framework.

## Core Responsibilities
- Register and unregister `MessageHandler` objects for specific message type IDs
- Route incoming `Message` objects to their registered handlers via type lookup
- Provide a default handler for message types without explicit registration
- Implement the `MessageHandler` interface to be composable (dispatcher-of-dispatchers)
- Support querying handlers with and without default fallback

## External Dependencies
- `<map>` ΓÇö STL container
- `Message.h` ΓÇö provides `Message` base class and `MessageTypeID` typedef
- `MessageHandler.h` ΓÇö provides `MessageHandler` interface
- `CommunicationsChannel` ΓÇö forward-referenced, defined elsewhere

# Source_Files/TCPMess/MessageHandler.h
## File Purpose
Defines an abstract message handler interface and provides template-based implementations for binding free functions and member methods as message handlers in a networking system. Enables type-safe, callback-style message dispatching.

## Core Responsibilities
- Define the abstract `MessageHandler` interface for polymorphic message handling
- Provide `TypedMessageHandlerFunction` template to wrap typed function pointers as handlers
- Provide `MessageHandlerMethod` template to wrap typed member methods as handlers
- Support automatic type casting from base `Message` and `CommunicationsChannel` to derived types
- Supply factory function to simplify handler method binding

## External Dependencies
- Forward declarations: `Message`, `CommunicationsChannel` (defined elsewhere)
- Standard library: `<cstdlib>` (minimal use; likely legacy include)

# Source_Files/TCPMess/MessageInflater.cpp
## File Purpose
Implements message deserialization using the prototype pattern. Reconstructs wire-format `UninflatedMessage` objects into fully-typed `Message` instances by cloning registered prototypes and populating them with data. Part of a TCP-based networking subsystem for a game engine.

## Core Responsibilities
- Deserialize uninflated messages into typed Message objects via prototype cloning and `inflateFrom()` 
- Register and manage prototype instances for each message type (factory registry)
- Provide graceful fallback when inflation fails (return cloned uninflated message)
- Manage memory of stored prototype clones
- Conditional compilation support (disabled when `DISABLE_NETWORKING` is defined)

## External Dependencies
- **`Message.h`**: defines `Message`, `UninflatedMessage`, and `MessageTypeID` (type aliases/classes defined elsewhere)
- **`Logging.h`**: provides logging macros `logWarning()`, `logAnomaly()`
- **`<map>`**: STL container for prototype registry
- **Conditional:** `#if !defined(DISABLE_NETWORKING)` wraps entire implementation

# Source_Files/TCPMess/MessageInflater.h
## File Purpose
Registry and factory for deserializing (inflating) network messages. Maintains prototype Message objects indexed by type ID and uses them to instantiate and populate concrete message instances from raw serialized data.

## Core Responsibilities
- Maintain a type-indexed registry of message prototypes
- Create concrete Message instances via prototype cloning and deserialization
- Register new message types by storing prototype instances
- Unregister message types and clean up allocated prototypes
- Provide lookup-by-type-ID for the inflation process

## External Dependencies
- `#include <map>` ΓÇö STL associative container for prototype registry
- `#include "Message.h"` ΓÇö Base Message class and UninflatedMessage class; also Message::clone(), Message::inflateFrom(), Message::type(), Message::~Message()
- SDL2 (transitively via Message.h) ΓÇö for Uint16, Uint8 type definitions


