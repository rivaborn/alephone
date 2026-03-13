# Source_Files/Lua/lua_ephemera.h

## File Purpose
Defines Lua binding classes for ephemera (temporary visual effects/objects) in the Aleph One game engine. Provides a Lua-accessible interface to create, access, and iterate over ephemera collections.

## Core Responsibilities
- Define `Lua_Ephemera` class template wrapping individual ephemera objects for Lua
- Define `Lua_Ephemeras` container template for iteration and collection access
- Declare registration function to expose ephemera classes to Lua state
- Bridge C++ ephemera objects with Lua scripting environment

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Ephemera` | typedef (L_Class specialization) | Lua userdata wrapper for individual ephemera with index-based object lookup |
| `Lua_Ephemeras` | typedef (L_Container specialization) | Lua container providing iteration, length, and indexed access over ephemera collection |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Ephemera_Name` | `extern char[]` | external | String constant `"ephemera"` used as Lua metatable/registry name |
| `Lua_Ephemeras_Name` | `extern char[]` | external | String constant `"Ephemera"` used as Lua container metatable/registry name |

## Key Functions / Methods

### Lua_Ephemera_register
- **Signature:** `int Lua_Ephemera_register(lua_State* L, const LuaMutabilityInterface& m)`
- **Purpose:** Register the ephemera Lua binding classes with the given Lua state, exposing both `Lua_Ephemera` and `Lua_Ephemeras` to scripts
- **Inputs:** 
  - `L`: Lua state to register classes with
  - `m`: Mutability interface controlling read/write access constraints in Lua bindings
- **Outputs/Return:** `int` (Lua return value, likely number of values pushed to stack or error code)
- **Side effects:** Modifies Lua registry by creating metatables, setting global container (`Ephemera`), registering metamethods
- **Calls:** Not inferable from this file (implementation in .cpp)
- **Notes:** Called during Lua environment initialization to expose ephemera scripting API

## Control Flow Notes
This is a **header-only declaration file**. Registration typically occurs during game initialization before scripts are loaded. The template inheritance (`L_Class` ΓåÆ `L_Container`) provides automatic support for:
- Individual ephemera access by index (`Ephemera[0]`)
- Container iteration (`for e in Ephemera() do ... end`)
- Length operator (`#Ephemera`)
- Metamethod dispatch for get/set operations

## External Dependencies
- **Lua 5.2 headers** (`lua.h`, `lauxlib.h`, `lualib.h`) ΓÇö scripting runtime
- **`lua_templates.h`** ΓÇö `L_Class<>` and `L_Container<>` template definitions
- **`cseries.h`** ΓÇö engine-wide types and macros
- **Undefined:** `LuaMutabilityInterface` class (defined elsewhere; controls access permissions)
