# Source_Files/TCPMess/MessageInflater.h

## File Purpose
Registry and factory for deserializing (inflating) network messages. Maintains prototype Message objects indexed by type ID and uses them to instantiate and populate concrete message instances from raw serialized data.

## Core Responsibilities
- Maintain a type-indexed registry of message prototypes
- Create concrete Message instances via prototype cloning and deserialization
- Register new message types by storing prototype instances
- Unregister message types and clean up allocated prototypes
- Provide lookup-by-type-ID for the inflation process

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MessageInflaterMap | typedef | std::map<MessageTypeID, Message*> mapping type IDs to prototype pointers |
| MessageInflater | class | Registry and factory for inflating serialized messages |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| mMap | MessageInflaterMap | instance (private member) | Stores message type ID ΓåÆ prototype Message* mappings |

## Key Functions / Methods

### inflate
- Signature: `Message* inflate(const UninflatedMessage& inSource)`
- Purpose: Deserialize a raw message buffer into a concrete typed Message instance
- Inputs: Reference to UninflatedMessage containing type ID and serialized bytes
- Outputs/Return: Dynamically allocated Message* of the appropriate concrete type
- Side effects: Allocates heap memory for the inflated message; caller must delete
- Calls: Implicitly calls Message::clone() and Message::inflateFrom() on the prototype (calls visible in implementation only)
- Notes: Assumes a prototype for inSource's type has been registered; behavior undefined if not found

### learnPrototype
- Signature: `void learnPrototype(const Message& inPrototype)`
- Purpose: Register a message prototype by extracting its type ID
- Inputs: Reference to a Message instance (any concrete subclass)
- Outputs/Return: None (void)
- Side effects: Inserts or overwrites entry in mMap; takes ownership of prototype pointer
- Calls: Calls inPrototype.type()
- Notes: Convenience wrapper; caller must ensure prototype lifetime

### learnPrototypeForType
- Signature: `void learnPrototypeForType(MessageTypeID inType, const Message& inPrototype)`
- Purpose: Explicitly register a prototype for a specific type ID
- Inputs: Type ID and reference to prototype Message
- Outputs/Return: None (void)
- Side effects: Inserts or overwrites entry in mMap; takes ownership of prototype pointer
- Calls: Direct map insertion
- Notes: Allows decoupling type ID from Message::type() if needed

### removePrototypeForType
- Signature: `void removePrototypeForType(MessageTypeID inType)`
- Purpose: Unregister a message type and deallocate its prototype
- Inputs: Type ID to remove
- Outputs/Return: None (void)
- Side effects: Removes entry from mMap; deletes prototype via delete operator
- Calls: std::map::erase()
- Notes: Safe to call if type not registered (no error thrown)

### ~MessageInflater
- Signature: `~MessageInflater()`
- Purpose: Destructor; clean up all registered prototypes
- Inputs: None
- Outputs/Return: None
- Side effects: Iterates mMap and calls delete on all stored Message* pointers
- Calls: std::map destructors; Message destructors
- Notes: Essential for preventing prototype memory leaks

## Control Flow Notes
Fits into a network message deserialization pipeline. Expected flow:
1. Application registers message prototypes (subclasses of Message) via learnPrototype/learnPrototypeForType
2. Network layer receives serialized data, wraps in UninflatedMessage
3. inflate() is called to reconstruct the typed message
4. Caller receives concrete Message subclass and can safely cast based on type ID

Destructor cleanup occurs at application shutdown or when MessageInflater instance is deleted.

## External Dependencies
- `#include <map>` ΓÇö STL associative container for prototype registry
- `#include "Message.h"` ΓÇö Base Message class and UninflatedMessage class; also Message::clone(), Message::inflateFrom(), Message::type(), Message::~Message()
- SDL2 (transitively via Message.h) ΓÇö for Uint16, Uint8 type definitions
