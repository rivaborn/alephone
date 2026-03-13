# Source_Files/Lua/lauxlib.h

## File Purpose
Header file declaring the Lua Auxiliary Library, which provides convenience functions and macros for C code building Lua libraries and interacting with the Lua stack. Includes argument validation, type conversion, buffer management, file handling, and library registration utilities.

## Core Responsibilities
- Declare auxiliary functions for parameter validation and type checking
- Provide type conversion utilities (strings, numbers, integers, userdata)
- Support metatable manipulation and userdata handling
- Implement string buffer building for efficient concatenation
- Manage file handles for Lua I/O operations
- Support library and module registration mechanisms
- Define convenience macros for common stack operations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `luaL_Reg` | struct | Function registry entry mapping a function name to a `lua_CFunction` pointer |
| `luaL_Buffer` | struct | Incremental string building buffer with embedded initial storage and linked Lua state |
| `luaL_Stream` | struct | File handle representation with stream pointer and close function |

## Global / File-Static State
None.

## Key Functions / Methods

### luaL_checkX (Argument Validators)
- **Signature:** `luaL_checklstring`, `luaL_checknumber`, `luaL_checkinteger`, `luaL_checkunsigned`, `luaL_checktype`, `luaL_checkany`, `luaL_checkudata`, `luaL_checkstack`
- **Purpose:** Validate that a stack argument has an expected type; raise an error if not.
- **Inputs:** `lua_State *L`, stack index (and expected type or constraints)
- **Outputs/Return:** Converted value (or void for type checks)
- **Side effects:** Raises `luaL_error()` on type mismatch
- **Notes:** Called at entry point of C functions exposed to Lua to enforce contract.

### luaL_optX (Optional Arguments with Defaults)
- **Signature:** `luaL_optlstring`, `luaL_optnumber`, `luaL_optinteger`, `luaL_optunsigned`
- **Purpose:** Extract an optional stack argument with a fallback default value.
- **Inputs:** `lua_State *L`, stack index, default value
- **Outputs/Return:** Converted value or default if argument absent/nil
- **Calls:** Uses `lua_isnoneornil()` to test argument presence
- **Notes:** Implements "Lua-style" optional parameters.

### Metatable Functions
- **`luaL_newmetatable`** ΓÇö Create and register a new metatable; returns 1 if new, 0 if exists.
- **`luaL_setmetatable`** ΓÇö Attach metatable (by registered name) to top-of-stack object.
- **`luaL_getmetafield`** ΓÇö Retrieve a field from an object's metatable (e.g., `__tostring`).
- **`luaL_callmeta`** ΓÇö Invoke a metamethod on an object (e.g., custom equality).
- **`luaL_testudata` / `luaL_checkudata`** ΓÇö Validate userdata against a registered metatable type.

### Buffer Management
- **`luaL_buffinit`** ΓÇö Initialize a buffer struct for incremental string building.
- **`luaL_prepbuffsize` / `luaL_prepbuffer`** ΓÇö Reserve space in buffer; auto-expands if needed.
- **`luaL_addlstring` / `luaL_addstring`** ΓÇö Append string data to buffer.
- **`luaL_addvalue`** ΓÇö Pop top stack value and append its string form to buffer.
- **`luaL_pushresult` / `luaL_pushresultsize`** ΓÇö Finalize buffer and push result string onto stack.
- **Signature:** Functions take `luaL_Buffer *B` and return `char*` or `void`.
- **Side effects:** Allocate/reallocate internal memory; modify stack.
- **Notes:** Macros `luaL_addchar()` and `luaL_addsize()` for direct buffer manipulation.

### Script Loading
- **`luaL_loadfilex`** ΓÇö Load Lua code from a file with binary/text mode.
- **`luaL_loadbufferx`** ΓÇö Load Lua code from memory buffer with mode.
- **`luaL_loadstring`** ΓÇö Load Lua code from C string.
- **Outputs/Return:** `LUA_OK` (0) on success, or error code (`LUA_ERRSYNTAX`, `LUA_ERRFILE`, etc.).
- **Notes:** Code is loaded but not executed; use `lua_pcall()` to run.

### Library Registration
- **`luaL_setfuncs`** ΓÇö Register array of `luaL_Reg` functions into a table.
- **`luaL_newlib`** ΓÇö Create a new library table and register functions (macro combines `luaL_newlibtable` + `luaL_setfuncs`).
- **`luaL_requiref`** ΓÇö Register a module loader function (compatibility).
- **Inputs:** `lua_State *L`, `luaL_Reg` array, upvalue count
- **Side effects:** Modify stack, populate Lua tables.

### Utility Functions
- **`luaL_error`** ΓÇö Format and raise an error with printf-style message.
- **`luaL_argerror`** ΓÇö Raise a formatted argument-validation error.
- **`luaL_where`** ΓÇö Push current source location (file:line) onto stack.
- **`luaL_ref` / `luaL_unref`** ΓÇö Create/destroy anchored references to stack values (for long-term storage).
- **`luaL_gsub`** ΓÇö String substitution (all occurrences of pattern `p` ΓåÆ replacement `r`).
- **`luaL_traceback`** ΓÇö Generate a stack traceback string.
- **`luaL_newstate`** ΓÇö Create a new Lua state (C-side allocation wrapper).
- **`luaL_checkversion`** ΓÇö Verify Lua library version match (macro, calls `luaL_checkversion_`).

## Control Flow Notes
This header declares functions used throughout the CΓåöLua binding layer. Typical usage patterns:

1. **Library initialization:** `luaL_newlib(L, functions_array)` + `luaL_setfuncs()` register exported functions.
2. **Script loading:** `luaL_loadfilex()` or `luaL_loadstring()` ΓåÆ `lua_pcall()` to execute.
3. **Argument validation:** Entry point of each C function checks arguments with `luaL_checkX()`.
4. **String building:** Buffer APIs accumulate strings without repeated allocations, finalized with `luaL_pushresult()`.
5. **Error handling:** `luaL_error()` / `luaL_argerror()` propagate errors back to Lua.
6. **Reference management:** `luaL_ref()` / `luaL_unref()` preserve Lua values across C function calls.

Not tied to a specific frame/render cycle; instead, called on-demand during CΓåöLua interop.

## External Dependencies
- **`lua.h`** ΓÇö Core Lua API (types: `lua_State`, `lua_CFunction`, `lua_Number`, `lua_Integer`; functions: `lua_createtable`, `lua_pcall`, `lua_getfield`, etc.)
- **`<stddef.h>`** ΓÇö Standard C types (`size_t`, `NULL`)
- **`<stdio.h>`** ΓÇö File I/O (`FILE`)
- **`luaconf.h`** (via `lua.h`) ΓÇö Lua configuration constants and type definitions

**Macros/Constants from lua.h used in lauxlib.h:**
- Error codes: `LUA_ERRERR`, `LUA_ERRSYNTAX`, `LUA_ERRFILE`
- Stack indices: `LUA_REGISTRYINDEX`
- Return values: `LUA_OK`, `LUA_MULTRET`
- Type constants: `LUA_TSTRING`, `LUA_TNUMBER`, etc.
- Version: `LUA_VERSION_NUM`
