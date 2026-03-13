# Source_Files/TCPMess/MessageDispatcher.h

## File Purpose
Implements a message-routing dispatcher that maps message type IDs to handler objects. Routes incoming messages to the appropriate handler based on type, with support for a default fallback handler. Acts as the central message router in a network communication framework.

## Core Responsibilities
- Register and unregister `MessageHandler` objects for specific message type IDs
- Route incoming `Message` objects to their registered handlers via type lookup
- Provide a default handler for message types without explicit registration
- Implement the `MessageHandler` interface to be composable (dispatcher-of-dispatchers)
- Support querying handlers with and without default fallback

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MessageDispatcherMap` | typedef | STL map from MessageTypeID to MessageHandler* for O(log n) type lookup |

## Global / File-Static State
None.

## Key Functions / Methods

### setHandlerForType
- **Signature:** `void setHandlerForType(MessageHandler* inHandler, MessageTypeID inType)`
- **Purpose:** Register a handler for a message type, or unregister if handler is NULL
- **Inputs:** Handler object (or NULL to clear), message type ID
- **Outputs/Return:** None
- **Side effects:** Modifies `mMap`; if NULL handler passed, delegates to `clearHandlerForType()`
- **Calls:** `clearHandlerForType()` (conditional)
- **Notes:** NULL handler is treated as unregister request, not as a valid handler

### handlerForType
- **Signature:** `MessageHandler* handlerForType(MessageTypeID inType)`
- **Purpose:** Look up handler for a type; return default handler if not found
- **Inputs:** Message type ID
- **Outputs/Return:** Handler object (or `mDefaultHandler` if type not registered)
- **Side effects:** None
- **Calls:** None
- **Notes:** Always returns non-NULL if a default handler is set; returns NULL only if type unregistered and no default

### handlerForTypeNoDefault
- **Signature:** `MessageHandler* handlerForTypeNoDefault(MessageTypeID inType)`
- **Purpose:** Look up handler for a type without fallback to default
- **Inputs:** Message type ID
- **Outputs/Return:** Handler object, or NULL if type not registered
- **Side effects:** None
- **Calls:** None

### clearHandlerForType
- **Signature:** `void clearHandlerForType(MessageTypeID inType)`
- **Purpose:** Remove handler registration for a message type
- **Inputs:** Message type ID
- **Outputs/Return:** None
- **Side effects:** Removes entry from `mMap`

### setDefaultHandler / defaultHandler
- **Purpose:** Set/get the fallback handler used when message type is not registered
- **Notes:** Default handler is optional; NULL is a valid value (no default)

### handle
- **Signature:** `void handle(Message* inMessage, CommunicationsChannel* inChannel)`
- **Purpose:** Dispatch an incoming message to its registered handler (implements `MessageHandler` interface)
- **Inputs:** Message object, communications channel
- **Outputs/Return:** None
- **Side effects:** Delegates to handler's `handle()` method; if no handler found, silently drops message
- **Calls:** `handlerForType()`, then target handler's `handle()`
- **Notes:** Silent no-op if handler is NULL; this is the entry point for routing

## Control Flow Notes
Part of a message-dispatch architecture: incoming messages flow into `handle()`, which performs type-based lookup and delegates to specific handlers. Fits into a comm loop as the central routing/multiplexing stage. Hierarchical composition is supported (a dispatcher can itself be a handler registered in another dispatcher).

## External Dependencies
- `<map>` ΓÇö STL container
- `Message.h` ΓÇö provides `Message` base class and `MessageTypeID` typedef
- `MessageHandler.h` ΓÇö provides `MessageHandler` interface
- `CommunicationsChannel` ΓÇö forward-referenced, defined elsewhere
