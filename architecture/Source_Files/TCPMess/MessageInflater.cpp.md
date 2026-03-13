# Source_Files/TCPMess/MessageInflater.cpp

## File Purpose
Implements message deserialization using the prototype pattern. Reconstructs wire-format `UninflatedMessage` objects into fully-typed `Message` instances by cloning registered prototypes and populating them with data. Part of a TCP-based networking subsystem for a game engine.

## Core Responsibilities
- Deserialize uninflated messages into typed Message objects via prototype cloning and `inflateFrom()` 
- Register and manage prototype instances for each message type (factory registry)
- Provide graceful fallback when inflation fails (return cloned uninflated message)
- Manage memory of stored prototype clones
- Conditional compilation support (disabled when `DISABLE_NETWORKING` is defined)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MessageInflaterMap` | typedef | `std::map<MessageTypeID, Message*>` ΓÇö associates message type IDs with prototype Message clones |

## Global / File-Static State
None.

## Key Functions / Methods

### inflate
- Signature: `Message* inflate(const UninflatedMessage& inSource)`
- Purpose: Deserialize an uninflated wire-format message into a fully-typed Message instance
- Inputs: `inSource` ΓÇö uninflated message containing type ID and wire data
- Outputs/Return: Pointer to inflated Message, or clone of inSource if inflation fails; never NULL
- Side effects: May allocate/deallocate Message objects; logs warnings/anomalies on failures
- Calls: `UninflatedMessage::inflatedType()`, `Message::clone()`, `Message::inflateFrom()`, `logWarning()`, `logAnomaly()`, `delete`
- Notes: Uses exception-safe design with try-catch. Fallback behavior ensures a message is always returned (either properly inflated or as uninflated clone). Handles 5 failure scenarios: missing type in registry, clone failure, exception in clone, inflateFrom returns false, exception in inflateFrom.

### learnPrototypeForType
- Signature: `void learnPrototypeForType(MessageTypeID inType, const Message& inPrototype)`
- Purpose: Register a prototype instance for a message type; used as a factory template
- Inputs: `inType` ΓÇö message type identifier; `inPrototype` ΓÇö template Message instance
- Outputs/Return: void
- Side effects: Clones inPrototype, stores in mMap, deletes any previous prototype for inType
- Calls: `Message::clone()`, `delete`
- Notes: Replaces existing prototype if type already registered

### removePrototypeForType
- Signature: `void removePrototypeForType(MessageTypeID inType)`
- Purpose: Unregister and free a message type prototype
- Inputs: `inType` ΓÇö message type identifier
- Outputs/Return: void
- Side effects: Deletes stored prototype, removes entry from mMap
- Calls: `delete`, `mMap.erase()`
- Notes: Safe no-op if type not found

### ~MessageInflater
- Signature: `~MessageInflater()`
- Purpose: Destructor; clean up all stored prototypes
- Side effects: Iterates mMap and deletes each prototype pointer
- Calls: `delete`

## Control Flow Notes
This appears to be a runtime service (singleton-like usage is likely). Initialization occurs via repeated calls to `learnPrototypeForType()` to populate the registry. The `inflate()` method is called during message reception to reconstruct typed messages from the wire. Cleanup happens on shutdown via destructor.

## External Dependencies
- **`Message.h`**: defines `Message`, `UninflatedMessage`, and `MessageTypeID` (type aliases/classes defined elsewhere)
- **`Logging.h`**: provides logging macros `logWarning()`, `logAnomaly()`
- **`<map>`**: STL container for prototype registry
- **Conditional:** `#if !defined(DISABLE_NETWORKING)` wraps entire implementation
