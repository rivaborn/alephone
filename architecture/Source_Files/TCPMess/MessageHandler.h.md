# Source_Files/TCPMess/MessageHandler.h

## File Purpose
Defines an abstract message handler interface and provides template-based implementations for binding free functions and member methods as message handlers in a networking system. Enables type-safe, callback-style message dispatching.

## Core Responsibilities
- Define the abstract `MessageHandler` interface for polymorphic message handling
- Provide `TypedMessageHandlerFunction` template to wrap typed function pointers as handlers
- Provide `MessageHandlerMethod` template to wrap typed member methods as handlers
- Support automatic type casting from base `Message` and `CommunicationsChannel` to derived types
- Supply factory function to simplify handler method binding

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MessageHandler` | abstract class | Base interface for all message handler implementations |
| `TypedMessageHandlerFunction` | template class | Adapts a typed function pointer into a `MessageHandler` |
| `MessageHandlerMethod` | template class | Adapts a typed member method into a `MessageHandler` |
| `MessageHandlerFunction` | typedef | Convenience alias for function handlers with base `Message` type |

## Global / File-Static State
None.

## Key Functions / Methods

### `MessageHandler::handle`
- Signature: `virtual void handle(Message* inMessage, CommunicationsChannel* inChannel) = 0`
- Purpose: Pure virtual method to process an incoming message on a communication channel
- Inputs: Pointer to message, pointer to channel
- Outputs/Return: None
- Side effects: Implementation-dependent
- Calls: (pure virtual)
- Notes: Base interface for all handler implementations

### `TypedMessageHandlerFunction::handle`
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)`
- Purpose: Invokes the wrapped function pointer with dynamically cast arguments
- Inputs: Base `Message*` and `CommunicationsChannel*`
- Outputs/Return: None (delegates to wrapped function)
- Side effects: Calls wrapped function if non-null
- Calls: User-supplied function via `mFunction`
- Notes: Performs `dynamic_cast` to convert base pointers to template types; silently ignores null function pointer

### `TypedMessageHandlerFunction::function / setFunction`
- Signature: `FunctionType function() const` / `void setFunction(FunctionType inFunction)`
- Purpose: Getter/setter for the wrapped function pointer
- Inputs: (setter) typed function pointer
- Outputs/Return: (getter) returns function pointer
- Side effects: None
- Calls: None
- Notes: Allows runtime modification of the callback

### `MessageHandlerMethod::handle`
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)`
- Purpose: Invokes the wrapped member method on the target object with dynamically cast arguments
- Inputs: Base `Message*` and `CommunicationsChannel*`
- Outputs/Return: None (delegates to wrapped method)
- Side effects: Calls method on `mObject` if both object and method are non-null
- Calls: User-supplied member method via `mMethod`
- Notes: Performs `dynamic_cast` to convert base pointers to template types; silently ignores null object or method pointer

### `newMessageHandlerMethod`
- Signature: `static inline MessageHandlerMethod<tTargetClass, tMessage, tChannel>* newMessageHandlerMethod(...)`
- Purpose: Factory function to allocate and initialize a typed message handler bound to a member method
- Inputs: Target object pointer and pointer-to-member-method
- Outputs/Return: Heap-allocated `MessageHandlerMethod` instance
- Side effects: Allocates memory with `new`
- Calls: `MessageHandlerMethod` constructor
- Notes: Template deduction simplifies syntax when creating handlers; caller responsible for managing returned pointer

## Control Flow Notes
This file defines the callback mechanism for a message dispatch system. Handlers are likely registered with a dispatcher/router and invoked when messages arrive on channels. The templates enable type-safe binding while the base class allows heterogeneous collections of handlers in dispatch tables.

## External Dependencies
- Forward declarations: `Message`, `CommunicationsChannel` (defined elsewhere)
- Standard library: `<cstdlib>` (minimal use; likely legacy include)
