# Source_Files/Lua/lua_serialize.h

## File Purpose
Declares the serialization interface for Lua objects in the game engine. Provides stream-based persistence functions to save and restore Lua state to/from buffers, enabling game save/load functionality.

## Core Responsibilities
- Export serialization API for Lua object persistence
- Provide stream abstraction via C++ standard library (`std::streambuf`)
- Wrap Lua C API for object marshalling

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### lua_save
- Signature: `bool lua_save(lua_State *L, std::streambuf* sb)`
- Purpose: Serialize the object on top of the Lua stack to a stream buffer
- Inputs: Active Lua state; stream buffer pointer
- Outputs/Return: Boolean (success/failure)
- Side effects: Reads stack top; writes binary data to stream
- Calls: Not visible in header (implementation elsewhere)
- Notes: Caller must push object onto stack before invoking

### lua_restore
- Signature: `bool lua_restore(lua_State *L, std::streambuf* sb)`
- Purpose: Deserialize an object from stream and push result onto Lua stack
- Inputs: Active Lua state; stream buffer pointer
- Outputs/Return: Boolean (success/failure); object pushed to stack on success
- Side effects: Modifies Lua stack; consumes bytes from stream
- Calls: Not visible in header (implementation elsewhere)
- Notes: Result appears at stack top; caller must check return value

## Control Flow Notes
Part of a save/load subsystem. Typical usage:
1. Prepare serializable Lua object and push to stack
2. Call `lua_save()` to persist to disk/buffer
3. Later, call `lua_restore()` to reload into a fresh or existing Lua state

## External Dependencies
- `cseries.h` ΓÇö Project common header
- Lua 5.2 C API: `lua.h`, `lauxlib.h`, `lualib.h`
- C++ standard: `<streambuf>` (abstract I/O stream interface)
