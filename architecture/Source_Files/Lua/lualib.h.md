# Source_Files/Lua/lualib.h

## File Purpose
Public header declaring the Lua standard library module initialization API. Provides function signatures and constants for loading built-in libraries (coroutine, table, I/O, OS, string, bitwise, math, debug, package) into a Lua state.

## Core Responsibilities
- Declare module initialization functions (`luaopen_*`) for each standard library
- Define canonical module name constants (e.g., `LUA_TABLIBNAME`, `LUA_MATHLIBNAME`)
- Declare bulk initialization function (`luaL_openlibs`) to register all libraries at once
- Provide assertion macro for runtime validation
- Include the core Lua API header (`lua.h`)

## Key Types / Data Structures
None. (All types used are defined in `lua.h`: `lua_State`)

## Global / File-Static State
None.

## Key Functions / Methods

### luaopen_base
- Signature: `int (luaopen_base) (lua_State *L)`
- Purpose: Initialize and register the Lua base library (global functions like `print`, `assert`, `type`, etc.)
- Inputs: `L` ΓÇö Lua state
- Outputs/Return: Integer status (implementation-defined)
- Side effects: Registers base functions into the Lua global environment
- Calls: (declared only; implementation in separate module)
- Notes: Must be called to enable core Lua functionality

### luaopen_coroutine, luaopen_table, luaopen_io, luaopen_os, luaopen_string, luaopen_bit32, luaopen_math, luaopen_debug, luaopen_package
- Signature: `int (luaopen_*) (lua_State *L)` (one for each library)
- Purpose: Initialize and register a specific Lua standard library
- Inputs: `L` ΓÇö Lua state
- Outputs/Return: Integer status
- Side effects: Registers library functions into Lua global environment (or package)
- Calls: (declared only; implementation in separate module)
- Notes: Each corresponds to a module name constant (e.g., `luaopen_table` Γåö `LUA_TABLIBNAME` = `"table"`)

### luaL_openlibs
- Signature: `void (luaL_openlibs) (lua_State *L)`
- Purpose: Convenience function to register all standard libraries at once
- Inputs: `L` ΓÇö Lua state
- Outputs/Return: (void)
- Side effects: Calls all `luaopen_*` functions to populate the global environment with all stdlib modules
- Calls: (declared only; typically calls all module initialization functions)
- Notes: Simplifies state initialization; alternative to selectively calling individual `luaopen_*` functions

## Control Flow Notes
This header is consumed during Lua engine initialization. Typical flow:
1. Create a new Lua state (`lua_newstate`)
2. Call `luaL_openlibs(L)` or selectively open specific libraries via `luaopen_*`
3. Proceed with script loading and execution

Game engines embedding Lua would call these during startup to populate the scripting environment.

## External Dependencies
- **`lua.h`**: Defines core Lua API types and functions (`lua_State`, `LUAMOD_API`, `LUALIB_API` macros, etc.)
- All module implementations ("defined elsewhere")
