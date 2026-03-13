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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Memento` | class (abstract) | Client-subclassable context storage for channel metadata |
| `CommunicationsChannel` | class | Primary TCP communication manager; manages sockets, queues, handlers |
| `CommunicationsChannelFactory` | class | Factory for server-side: listens on port and creates `CommunicationsChannel` for each accepted connection |
| `CommunicationResult` (private enum) | enum | Internal result code: `kIncomplete`, `kComplete`, `kError` |

## Global / File-Static State
None.

## Key Functions / Methods

### CommunicationsChannel constructors
- **Signature:** `CommunicationsChannel()` (default); `CommunicationsChannel(std::unique_ptr<TCPsocket> inSocket)` (from socket)
- **Purpose:** Create a new channel, either unconnected or wrapping an existing TCP socket.
- **Inputs:** Optional existing socket.
- **Outputs/Return:** Object initialized; `mConnected` reflects socket validity.
- **Side effects:** Initializes all queues, buffers, and activity timestamps.

### pump()
- **Signature:** `void pump()`
- **Purpose:** Move data between TCP socket and internal buffers without invoking handlers; idempotent.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `pumpReceivingSide()` and `pumpSendingSide()` to shift raw bytes; updates `mTicksAtLastReceive/Send`.
- **Notes:** Does *not* dispatch message handlers; used by frame loop for background I/O.

### dispatchOneIncomingMessage()
- **Signature:** `bool dispatchOneIncomingMessage()`
- **Purpose:** Pop one message from `mIncomingMessages` and invoke the registered `MessageHandler` callback.
- **Inputs:** None.
- **Outputs/Return:** `true` if a message was dispatched; `false` if queue empty.
- **Side effects:** Invokes `mMessageHandler` callback; removes message from queue.
- **Notes:** Handler is responsible for message deletion.

### dispatchIncomingMessages()
- **Signature:** `void dispatchIncomingMessages()`
- **Purpose:** Dispatch all queued incoming messages via `dispatchOneIncomingMessage()` loop.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Empties `mIncomingMessages`; calls handler for each message.

### receiveMessage()
- **Signature:** `Message* receiveMessage(Uint32 inOverallTimeout = kSSRAnyMessageTimeout, Uint32 inInactivityTimeout = kSSRAnyDataTimeout)`
- **Purpose:** Blocking synchronous receive: pump and dispatch until one message arrives or timeout/disconnect.
- **Inputs:** Overall timeout (ms, default 30s); inactivity timeout (ms, default 10s).
- **Outputs/Return:** Pointer to inflated `Message` (caller must delete); `NULL` on timeout or disconnect.
- **Side effects:** Pumps, dispatches other messages via handler, blocks thread.
- **Calls:** `pump()`, `dispatchIncomingMessages()`, waits on internal buffering.
- **Notes:** "Overall" limits total wait time; "inactivity" limits idle time. Distinction allows for slow/retransmit scenarios.

### receiveSpecificMessage() (template overloads)
- **Signature:** 
  - `Message* receiveSpecificMessage(MessageTypeID inType, Uint32 inOverallTimeout, Uint32 inInactivityTimeout)`
  - `template<typename tMessage> tMessage* receiveSpecificMessage(MessageTypeID, ...)`
  - `template<typename tMessage> tMessage* receiveSpecificMessage(...)`  (deduces type from template arg)
- **Purpose:** Synchronous receive filtering: pump until message of `inType` arrives; handle other types normally.
- **Inputs:** Message type ID; timeouts.
- **Outputs/Return:** Pointer to message of expected type (or `NULL`); non-matching messages dispatched to handler.
- **Side effects:** Same as `receiveMessage()` plus handler dispatch for filtered messages.
- **Notes:** Generic overload uses `tMessage::kType` static member. Template version performs `dynamic_cast<tMessage*>` and `release()` to transfer ownership.

### receiveSpecificMessageOrThrow()
- **Signature:** `template<typename tMessage> tMessage* receiveSpecificMessageOrThrow(Uint32 inOverallTimeout, Uint32 inInactivityTimeout)`
- **Purpose:** Type-safe synchronous receive; throws `FailedToReceiveSpecificMessageException` if type mismatch or timeout.
- **Inputs:** Timeouts.
- **Outputs/Return:** Pointer to expected message type; never `NULL` on success.
- **Side effects:** Throws exception on failure.

### enqueueOutgoingMessage()
- **Signature:** `void enqueueOutgoingMessage(const Message& inMessage)`
- **Purpose:** Queue a message for transmission; copies message bytes.
- **Inputs:** Reference to message to send.
- **Outputs/Return:** None.
- **Side effects:** Calls `deflate()` on message, appends to `mOutgoingMessages` queue.
- **Notes:** Caller retains ownership of input message.

### flushOutgoingMessages()
- **Signature:** `void flushOutgoingMessages(bool dispatchIncomingMessages, Uint32 inOverallTimeout = kOutgoingOverallTimeout, Uint32 inInactivityTimeout = kOutgoingInactivityTimeout)`
- **Purpose:** Blocking send: pump outgoing queue until empty or timeout. Optionally dispatch incoming.
- **Inputs:** Flag to dispatch incoming messages; timeouts (default 10 min overall, 10s inactivity).
- **Outputs/Return:** None (blocks).
- **Side effects:** Empties `mOutgoingMessages` via `pumpSendingSide()`; optionally calls `dispatchIncomingMessages()`.
- **Notes:** Used before critical operations (e.g., before disconnect).

### multipleFlushOutgoingMessages() (static)
- **Signature:** `static void multipleFlushOutgoingMessages(std::vector<CommunicationsChannel*>&, bool dispatchIncomingMessages, Uint32 inOverallTimeout, Uint32 inInactivityTimeout)`
- **Purpose:** Efficient batch flush: wait until all channels have emptied outgoing queues (avoids per-channel blocking).
- **Inputs:** Vector of channel pointers; dispatch flag; timeouts.
- **Outputs/Return:** None.
- **Side effects:** Pumps all channels in loop; dispatches if enabled.

### connect() (overloads)
- **Signature:** 
  - `void connect(const std::string& inAddressString, Uint16 inPort)` ΓÇö address as hostname/IP string
  - `void connect(const IPaddress& inAddress)` ΓÇö pre-parsed address
- **Purpose:** Establish outgoing TCP connection to remote peer.
- **Inputs:** Address (string or `IPaddress` object); port (host byte order for string overload; embedded in `IPaddress` for other).
- **Outputs/Return:** None.
- **Side effects:** Creates/replaces `mSocket`; sets `mConnected = true` on success; failure sets `mConnected = false`.

### disconnect()
- **Signature:** `void disconnect()`
- **Purpose:** Close the TCP connection.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Closes socket; sets `mConnected = false`.

### Activity tracking accessors
- **ticksAtLastReceive()**, **ticksAtLastSend()** ΓÇö Return machine tick count at last I/O.
- **millisecondsSinceLastReceive()**, **millisecondsSinceLastSend()** ΓÇö Compute elapsed time since activity (via `machine_tick_count() - mTicks*`).
- **Purpose:** Allow caller to monitor channel liveness/heartbeat.

## Control Flow Notes

**Outgoing path (sending):**
1. Application calls `enqueueOutgoingMessage(msg)` ΓåÆ queued to `mOutgoingMessages`
2. `pump()` or `pumpSendingSide()` invoked (from game loop) ΓåÆ `send_some()` writes to TCP
3. `flushOutgoingMessages()` (before disconnect/shutdown) blocks until queue empty

**Incoming path (receiving):**
1. `pump()` or `pumpReceivingSide()` ΓåÆ `receive_some()` reads raw bytes from TCP ΓåÆ accumulate in `mIncomingHeader` and `mIncomingMessage`
2. Once header/message complete, inflated and appended to `mIncomingMessages` queue
3. `dispatchIncomingMessages()` pops from queue, invokes `mMessageHandler` callback
4. Alternatively, `receiveMessage()` / `receiveSpecificMessage()` blocks until desired message arrives

**Activity tracking:**
- `mTicksAtLastReceive` and `mTicksAtLastSend` updated during `pump()` for timeout detection.

## External Dependencies
- **STL headers:** `<list>`, `<string>`, `<memory>`, `<stdexcept>`, `<vector>`
- **NetworkInterface.h:** `TCPsocket`, `TCPlistener`, `IPaddress`
- **Message.h:** `Message`, `UninflatedMessage`, `MessageTypeID`
- **csmisc.h:** `machine_tick_count()` for timing
- **Forward declarations:** `MessageInflater` (inflates raw bytes to `Message`); `MessageHandler` (callback interface for dispatched messages)
