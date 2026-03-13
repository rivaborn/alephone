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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CommunicationsChannel | class | Main bidirectional messaging channel over TCP |
| CommunicationsChannelFactory | class | Accepts incoming TCP connections and wraps them as channels |
| Memento | class | Polymorphic base for storing user data with a channel |
| MessageQueue | typedef (std::list<Message*>) | Queue of received/processed messages |
| UninflatedMessageQueue | typedef (std::list<UninflatedMessage*>) | Queue of outgoing uninflated messages |
| CommunicationResult | enum | Result of partial send/receive: kIncomplete, kComplete, kError |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| kMaximumMessageLength | const int | file-static | Reject any incoming message claiming >4MB; prevents DoS |
| kSSRPumpInterval | const int | file-static | Sleep duration (50ms) between pump calls in synchronous receive |
| kFlushPumpInterval | const int | file-static | Sleep duration (50ms) between pump calls when flushing outgoing |
| kHeaderPackedSize | const int | class-static | Message header size in bytes (8) |
| kHeaderMagic | const uint16 | class-static | Header magic constant (0xDEAD) for validation |

## Key Functions / Methods

### receive_some
- Signature: `CommunicationResult receive_some(byte* inBuffer, size_t& ioBufferPosition, size_t inBufferLength)`
- Purpose: Attempt to receive up to `inBufferLength - ioBufferPosition` bytes into buffer
- Inputs: target buffer, current position (in/out), total buffer length
- Outputs/Return: kIncomplete (got some/none), kComplete (filled buffer), kError (socket error)
- Side effects: Updates `ioBufferPosition` on success; updates `mTicksAtLastReceive`; calls `disconnect()` on error
- Calls: `mSocket->receive()`
- Notes: Non-blocking; returns immediately if zero bytes available

### send_some
- Signature: `CommunicationResult send_some(byte* inBuffer, size_t& ioBufferPosition, size_t inBufferLength)`
- Purpose: Attempt to send `inBufferLength - ioBufferPosition` bytes from buffer
- Inputs: source buffer, current position (in/out), total buffer length
- Outputs/Return: kIncomplete (sent some/none), kComplete (sent all), kError (socket error)
- Side effects: Updates `ioBufferPosition` on success; updates `mTicksAtLastSend`; calls `disconnect()` on error
- Calls: `mSocket->send()`
- Notes: Non-blocking; returns immediately if TCP would block

### receiveHeader
- Signature: `bool receiveHeader()`
- Purpose: Receive an 8-byte message header; parse and validate it; prepare for message body reception
- Inputs: None (uses internal `mIncomingHeader`, `mIncomingHeaderPosition`)
- Outputs/Return: true if should continue pumping, false if blocked/incomplete
- Side effects: Allocates new `UninflatedMessage` on successful header; calls `disconnect()` on invalid magic/length; resets `mIncomingHeaderPosition`
- Calls: `receive_some()`, `AIStreamBE` >> operator, `new UninflatedMessage()`
- Notes: Header format is magic (uint16 BE), type (uint16 BE), length (uint32 BE, includes header); length is validated against `kMaximumMessageLength`

### _receiveMessage
- Signature: `bool _receiveMessage()`
- Purpose: Receive message body; inflate if inflater is set; enqueue; prepare for next header
- Inputs: None (uses internal `mIncomingMessage`, `mIncomingMessagePosition`)
- Outputs/Return: true if should continue pumping, false if blocked/incomplete
- Side effects: Deletes original `UninflatedMessage` if inflated; enqueues inflated or uninflated message; resets position state
- Calls: `receive_some()`, `mMessageInflater->inflate()`, `mIncomingMessages.push_back()`
- Notes: Optional inflation; ownership transferred to queue

### sendHeader
- Signature: `bool sendHeader()`
- Purpose: Send the queued message's 8-byte header (magic, type, length)
- Inputs: None (uses internal `mOutgoingHeader`, `mOutgoingHeaderPosition`)
- Outputs/Return: true if should continue pumping, false if blocked/incomplete
- Side effects: Resets `mOutgoingMessagePosition` on completion
- Calls: `send_some()`

### sendMessage
- Signature: `bool sendMessage()`
- Purpose: Send the message body of the front outgoing message
- Inputs: None (uses internal queue, `mOutgoingMessagePosition`)
- Outputs/Return: true if should continue pumping, false if blocked/incomplete
- Side effects: Dequeues and deletes message on completion; resets `mOutgoingHeaderPosition`
- Calls: `send_some()`, `delete`, `mOutgoingMessages.pop_front()`

### pumpSendingSide
- Signature: `void pumpSendingSide()`
- Purpose: Drive the sending state machine: fill header buffer, send header, send body, repeat for all queued messages
- Inputs: None (uses internal queues and state)
- Outputs/Return: None
- Side effects: Modifies header buffer and position; dequeues messages; calls `disconnect()` on error
- Calls: `sendHeader()`, `sendMessage()`, `AOStreamBE` << operator
- Notes: Loops until blocked or queue empty; builds header on-demand when `mOutgoingHeaderPosition == 0`

### pumpReceivingSide
- Signature: `void pumpReceivingSide()`
- Purpose: Drive the receiving state machine: receive headers, receive bodies, repeat until blocked
- Inputs: None (uses internal state)
- Outputs/Return: None
- Side effects: Allocates/enqueues messages; calls `disconnect()` on error
- Calls: `receiveHeader()`, `_receiveMessage()`
- Notes: Loops until blocked or disconnected

### pump
- Signature: `void pump()`
- Purpose: Non-blocking I/O pump; process both sending and receiving sides once
- Inputs: None
- Outputs/Return: None
- Side effects: May send/receive data, enqueue/dequeue messages, update timestamps, or disconnect
- Calls: `pumpSendingSide()`, `pumpReceivingSide()`
- Notes: Safe to call repeatedly; used in both synchronous and asynchronous patterns

### receiveMessage
- Signature: `Message* receiveMessage(Uint32 inOverallTimeout = kSSRAnyMessageTimeout, Uint32 inInactivityTimeout = kSSRAnyDataTimeout)`
- Purpose: Block until a message is received, overall timeout expires, inactivity timeout expires, or disconnection
- Inputs: overall timeout (ms), inactivity timeout (ms, "no data" timeout)
- Outputs/Return: Pointer to inflated Message; NULL if timeout/disconnect
- Side effects: Repeatedly calls `pump()`; sleeps `kSSRPumpInterval` between pumps; dequeues message
- Calls: `pump()`, `sleep_for_machine_ticks()`, `machine_tick_count()`
- Notes: Caller owns returned message; inactivity timeout uses max of current time and last receive time to allow long overall waits with short inactivity windows

### receiveSpecificMessage
- Signature: `Message* receiveSpecificMessage(MessageTypeID inType, Uint32 inOverallTimeout = kSSRSpecificMessageTimeout, Uint32 inInactivityTimeout = kSSRAnyDataTimeout)`
- Purpose: Block until a message of a specific type is received; handle (via callback) and discard other types
- Inputs: desired message type, overall timeout (ms), inactivity timeout (ms)
- Outputs/Return: Pointer to desired Message type; NULL if timeout/disconnect
- Side effects: Calls `pump()`; dispatches non-matching messages to handler; deletes non-matching messages
- Calls: `receiveMessage()`, `messageHandler()->handle()`, `delete`
- Notes: Fires message handler for undesired types; useful for skipping unsolicited messages

### flushOutgoingMessages
- Signature: `void flushOutgoingMessages(bool shouldDispatchIncomingMessages, Uint32 inOverallTimeout = kOutgoingOverallTimeout, Uint32 inInactivityTimeout = kOutgoingInactivityTimeout)`
- Purpose: Block until all outgoing messages are sent to TCP, or timeout/disconnect
- Inputs: whether to dispatch incoming messages during flush, overall timeout (ms), inactivity timeout (ms)
- Outputs/Return: None
- Side effects: Repeatedly calls `pump()` and optionally `dispatchIncomingMessages()`; sleeps `kFlushPumpInterval` between pumps
- Calls: `pump()`, `dispatchIncomingMessages()`, `sleep_for_machine_ticks()`
- Notes: Uses both overall and inactivity timeouts like `receiveMessage`; useful for ensuring delivery before shutdown

### multipleFlushOutgoingMessages
- Signature: `static void multipleFlushOutgoingMessages(std::vector<CommunicationsChannel*>&, bool shouldDispatchIncomingMessages, Uint32 inOverallTimeout, Uint32 inInactivityTimeout)`
- Purpose: Efficiently flush multiple channels in parallel; more efficient than calling `flushOutgoingMessages()` on each
- Inputs: vector of channels, dispatch flag, overall timeout (ms), inactivity timeout (ms)
- Outputs/Return: None
- Side effects: Pumps all channels until all queues empty or timeout
- Calls: `pump()`, `dispatchIncomingMessages()` on each channel
- Notes: Tracks which channels still have pending messages; sleeps once per iteration

### dispatchOneIncomingMessage
- Signature: `bool dispatchOneIncomingMessage()`
- Purpose: Deliver one queued received message to the handler callback (if installed) and delete it
- Inputs: None (uses internal queue)
- Outputs/Return: true if a message was dispatched; false if queue empty
- Side effects: Calls handler; deletes message; dequeues
- Calls: `messageHandler()->handle()`, `delete`, `pop_front()`

### dispatchIncomingMessages
- Signature: `void dispatchIncomingMessages()`
- Purpose: Deliver all queued received messages to handlers and delete them
- Inputs: None
- Outputs/Return: None
- Side effects: Calls handlers; deletes messages; empties queue
- Calls: `dispatchOneIncomingMessage()` in loop

### enqueueOutgoingMessage
- Signature: `void enqueueOutgoingMessage(const Message& inMessage)`
- Purpose: Queue a message for sending (copies/deflates the message)
- Inputs: Message reference
- Outputs/Return: None
- Side effects: Enqueues `UninflatedMessage`; does nothing if disconnected
- Calls: `inMessage.deflate()`, `push_back()`
- Notes: Safe to call only when connected; caller retains ownership of input message

### connect (IPaddress)
- Signature: `void connect(const IPaddress& inAddress)`
- Purpose: Establish outgoing TCP connection to a remote address
- Inputs: IPaddress (with port in network byte order)
- Outputs/Return: None
- Side effects: Asserts not already connected; clears any leftover state; sets non-blocking mode; updates timestamps
- Calls: `NetGetNetworkInterface()->tcp_connect_socket()`, `mSocket->set_non_blocking()`
- Notes: Resets all incoming/outgoing queues and buffers

### connect (string, port)
- Signature: `void connect(const std::string& inAddressString, uint16 inPort)`
- Purpose: Establish outgoing TCP connection by hostname/port (convenience wrapper)
- Inputs: hostname string, port (host byte order)
- Outputs/Return: None
- Side effects: Calls the IPaddress variant if resolution succeeds
- Calls: `NetGetNetworkInterface()->resolve_address()`, `connect(IPaddress)`
- Notes: Does nothing if DNS resolution fails

### disconnect
- Signature: `void disconnect()`
- Purpose: Close the TCP connection and discard all pending I/O
- Inputs: None
- Outputs/Return: None
- Side effects: Releases socket; clears flags and queues; resets position counters
- Calls: `mSocket.reset()`, `delete` on queued messages, `pop_front()`
- Notes: Safe to call multiple times; leaves channel ready for a new `connect()`

### peerAddress
- Signature: `IPaddress peerAddress() const`
- Purpose: Get the remote IP address of the connected peer
- Inputs: None
- Outputs/Return: IPaddress of peer
- Calls: `mSocket->remote_address()`
- Notes: Valid only while connected

### CommunicationsChannelFactory constructor
- Signature: `CommunicationsChannelFactory(uint16 inPort)`
- Purpose: Create a listener for accepting incoming TCP connections on a port
- Inputs: port number (host byte order)
- Outputs/Return: None
- Side effects: Creates non-blocking TCP listener
- Calls: `NetGetNetworkInterface()->tcp_open_listener()`, `set_non_blocking()`

### CommunicationsChannelFactory::newIncomingConnection
- Signature: `CommunicationsChannel* newIncomingConnection()`
- Purpose: Accept one pending incoming connection (non-blocking)
- Inputs: None
- Outputs/Return: Pointer to new CommunicationsChannel wrapping accepted socket; nullptr if no pending connection
- Side effects: Allocates channel; accepts and wraps socket
- Calls: `mSocketListener->accept_connection()`, `new CommunicationsChannel()`, `set_non_blocking()`
- Notes: Non-blocking; caller owns returned channel

**Notes on minor helpers:**
- `isConnected()`, `isMessageAvailable()`, `ticksAtLastReceive()`, `ticksAtLastSend()`, `millisecondsSinceLastReceive()`, `millisecondsSinceLastSend()`: simple getters
- `setMessageInflater()`, `messageInflater()`, `setMessageHandler()`, `messageHandler()`, `setMemento()`, `memento()`: accessors for callbacks and user data

## Control Flow Notes
**Asynchronous (frame-based):**
Main loop calls `pump()` each frame. Data flows incrementally; pump returns immediately whether or not data was available. Application processes received messages by calling `dispatchIncomingMessages()`.

**Synchronous (blocking):**
Functions like `receiveMessage()` and `flushOutgoingMessages()` call `pump()` in a loop with interleaved sleeps (`kSSRPumpInterval` = 50ms) until completion or timeout. These are useful for setup, shutdown, or critical-path operations.

**State machines:**
- **Receive side:** alternates between receiving header (8 bytes) and message body; validates header magic and length; optionally inflates; enqueues for later dispatch.
- **Send side:** fills header buffer on demand, sends header, sends body, dequeues; repeats for next message in queue.

Both state machines loop via `pumpSendingSide()` / `pumpReceivingSide()` until blocked (would-block on socket, empty queue, etc.), allowing multiple messages per pump call if TCP buffers permit.

## External Dependencies
- **"CommunicationsChannel.h"** ΓÇö class declarations and type definitions
- **"AStream.h"** ΓÇö AIStreamBE (big-endian input), AOStreamBE (big-endian output) for serializing header
- **"MessageInflater.h"** ΓÇö `MessageInflater::inflate()` for message decompression
- **"MessageHandler.h"** ΓÇö `MessageHandler` interface for receive callbacks
- **"network.h"** ΓÇö `NetGetNetworkInterface()` to obtain TCP socket factory
- **"cseries.h"** ΓÇö common engine types (`Uint8`, `Uint16`, `Uint32`), `machine_tick_count()`, `sleep_for_machine_ticks()`
- **<stdlib.h>, <iostream>, <cerrno>, <algorithm>** ΓÇö standard library
- **Defined elsewhere:** `TCPsocket`, `TCPlistener`, `IPaddress` (from NetworkInterface.h), `Message`, `UninflatedMessage`, `MessageTypeID` (from Message.h), `Memento` (forward decl.), `MessageInflater`, `MessageHandler`
