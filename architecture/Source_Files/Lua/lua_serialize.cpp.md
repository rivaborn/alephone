# Source_Files/Lua/lua_serialize.cpp

## File Purpose
Serializes and deserializes Lua objects to/from binary streams for game state persistence. Supports all core Lua types with reference deduplication to handle circular references and shared objects. Based on Pluto but with simplified design.

## Core Responsibilities
- Recursively serialize Lua values (nil, number, boolean, string, table, userdata) to binary format
- Recursively deserialize binary data back into Lua values with proper type reconstruction
- Maintain reference tables during save/restore to deduplicate objects and preserve identity
- Validate table keys and filter unsupported types (functions, light userdata)
- Handle userdata reconstruction via metatable `__new` callbacks
- Implement version checking for forward compatibility

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SAVED_REFERENCE_PSEUDOTYPE` | const (int) | Pseudo-type (-2) marking a reference to a previously saved object |
| `kVersion` | const (uint16) | Serialization format version (1) for compatibility checking |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SAVED_REFERENCE_PSEUDOTYPE` | const int | file-static | Type marker for object references |
| `kVersion` | const uint16 | file-static | Format version for saved data |

## Key Functions / Methods

### valid_key
- **Signature:** `static bool valid_key(int type)`
- **Purpose:** Determine if a Lua type is valid as a table key during serialization
- **Inputs:** Lua type constant (LUA_TNUMBER, LUA_TBOOLEAN, LUA_TSTRING, LUA_TTABLE, LUA_TUSERDATA)
- **Outputs/Return:** `true` if type is a valid key type; `false` otherwise
- **Side effects:** None
- **Calls:** None
- **Notes:** Rejects nil and function types; only these five types are serializable as keys

### save
- **Signature:** `static void save(lua_State *L, BOStreamBE& s, uint32& counter)`
- **Purpose:** Recursively serialize a Lua value from the stack to a binary stream
- **Inputs:** 
  - `L`: Lua state with value at top of stack
  - `s`: Output stream (big-endian format)
  - `counter`: Reference counter (incremented for tables/userdata)
- **Outputs/Return:** Void; writes to stream
- **Side effects:** Modifies stream; manipulates Lua stack (push/pop); updates reference table at `L[1]`
- **Calls:** Recursive `save()` calls; Lua C API stack operations (`lua_pushvalue`, `lua_rawget`, `lua_next`, `lua_getmetatable`)
- **Notes:** 
  - Checks reference table first; if object already saved, writes reference ID instead
  - For tables: adds reference, writes all key-value pairs (only valid keys), terminates with nil key
  - For userdata: adds reference, serializes metatable name and index field
  - Silently ignores unsupported types (functions, threads)

### lua_save
- **Signature:** `bool lua_save(lua_State *L, std::streambuf* sb)`
- **Purpose:** Public API to serialize the top-of-stack Lua object
- **Inputs:** 
  - `L`: Lua state with object at position 1 (requires exactly one object)
  - `sb`: C++ stream buffer for output
- **Outputs/Return:** `true` on success; `false` on stream failure
- **Side effects:** Writes version (uint16) and serialized data to stream; creates/destroys reference table; clears stack on error
- **Calls:** `save()`; logging via `logWarning()`
- **Notes:** Asserts stack depth is 1; reference table is internal and removed before return; catches `basic_bstream::failure` exceptions

### restore
- **Signature:** `static int restore(lua_State *L, BIStreamBE& s)`
- **Purpose:** Recursively deserialize a Lua value from binary stream and push onto stack
- **Inputs:** 
  - `L`: Lua state
  - `s`: Input stream (big-endian format)
- **Outputs/Return:** `int` (type tag of restored value; value is on top of stack)
- **Side effects:** Reads from stream; modifies stack and reference table; invokes userdata constructors
- **Calls:** Recursive `restore()` calls; Lua C API (`lua_pushnil`, `lua_newtable`, `lua_getfield`, `lua_call`, `lua_isfunction`)
- **Notes:** 
  - For tables: creates empty table, registers it, then restores k-v pairs until nil key
  - For userdata: reads metatable name and index, calls `__new` function from registry to reconstruct
  - For references: looks up previously restored object in reference table by ID
  - Skips invalid key-value pairs (e.g., nil keys from userdata restoration failures)

### lua_restore
- **Signature:** `bool lua_restore(lua_State *L, std::streambuf* sb)`
- **Purpose:** Public API to deserialize a Lua object from stream and push onto stack
- **Inputs:** 
  - `L`: Lua state
  - `sb`: C++ stream buffer for input
- **Outputs/Return:** `true` on success; `false` on version mismatch or stream failure
- **Side effects:** Reads version and data from stream; creates/destroys reference table; clears stack on error
- **Calls:** `restore()`; logging via `logWarning()`
- **Notes:** Checks that saved version Γëñ `kVersion`; logs warning if file is from newer version; catches exceptions; cleans up reference table before return

## Control Flow Notes
These functions are called during game save/load sequences. `lua_save()` is invoked when serializing game state (Lua tables/objects) to disk. `lua_restore()` is invoked when loading saved game state. Both manage an internal reference table (at stack position 1) that maps objects to unique IDs, enabling correct handling of circular references, duplicate references, and userdata reconstruction. The save/restore pattern is symmetric: a saved object can be faithfully restored with object identity and structure preserved.

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (stack manipulation, type checking, metatable access)
- **BStream.h:** `BOStreamBE`, `BIStreamBE` (big-endian binary I/O)
- **Logging.h:** `logWarning()` macro (error reporting)
- **cseries.h:** Base types (`uint32`, `uint16`, `uint8`, `int8`, `double`)
