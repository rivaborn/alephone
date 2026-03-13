# Source_Files/Lua/lua_projectiles.h

## File Purpose
Declares Lua/C++ binding wrappers for projectile objects and projectile type enumerations. Exposes these game entities to Lua scripts through the Lua VM via template-based wrapper classes and registration functions.

## Core Responsibilities
- Define Lua-accessible wrapper class for individual projectiles (indexing into game's projectile collection)
- Define Lua-accessible container class for iterating all projectiles
- Define Lua-accessible enum wrapper for projectile type classifications
- Define Lua-accessible container for projectile type enumerations (with string lookup via mnemonics)
- Declare registration function to bind these classes to the Lua VM

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Lua_Projectile | typedef (L_Class) | Wraps a projectile index for Lua access; supports property getters/setters |
| Lua_Projectiles | typedef (L_Container) | Collection wrapper; enables iteration and indexed access to all projectiles |
| Lua_ProjectileType | typedef (L_Enum) | Enum type for projectile classifications; supports numeric indices and string mnemonics |
| Lua_ProjectileTypes | typedef (L_EnumContainer) | Enum container; allows lookup by both numeric index and mnemonic string |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Lua_Projectile_Name | char[] | extern | String identifier ("projectile") used as Lua metatable name |
| Lua_Projectiles_Name | char[] | extern | String identifier ("Projectiles") for container global |
| Lua_ProjectileType_Name | char[] | extern | String identifier ("projectile_type") for enum type |
| Lua_ProjectileTypes_Name | char[] | extern | String identifier ("ProjectileTypes") for enum container global |

## Key Functions / Methods

### Lua_Projectiles_register
- **Signature**: `int Lua_Projectiles_register(lua_State *L, const LuaMutabilityInterface& m);`
- **Purpose**: Register projectile and projectile type classes/containers with the Lua VM, exposing them to Lua scripts.
- **Inputs**: 
  - `lua_State *L`: Lua virtual machine state
  - `const LuaMutabilityInterface& m`: Controls whether exposed objects are mutable from Lua
- **Outputs/Return**: Integer status (Lua convention: 0 or count of stack results)
- **Side effects**: Populates Lua registry with metatables, method tables, and container globals; modifies Lua stack
- **Calls**: Not inferable (implementation in .cpp file; likely calls L_Class::Register, L_Container::Register, L_Enum::Register, L_EnumContainer::Register)
- **Notes**: Implementation must set up all four wrapper types and register them before returning

## Control Flow Notes
This header is included by implementation files during game initialization. The registration function is called once to set up Lua bindings. Afterward, Lua scripts can:
- Iterate projectiles via `Projectiles()` (container iteration)
- Access individual projectile properties via `Projectiles[index]:property`
- Classify projectiles via `ProjectileType` enum (mnemonic string lookup supported)

## External Dependencies
- **Includes**: 
  - `cseries.h` ΓÇö platform abstraction, standard types
  - `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö Lua 5.2 C API
  - `lua_templates.h` ΓÇö template wrappers (L_Class, L_Container, L_Enum, L_EnumContainer)
- **Defined elsewhere**: 
  - `LuaMutabilityInterface` ΓÇö defined in lua_script.h (or similar); controls read/write access
  - Implementation of `Lua_Projectiles_register()` ΓÇö in a .cpp file (e.g., lua_projectiles.cpp)
  - Actual game projectile data structures ΓÇö in core game engine (likely projectiles.h/projectiles.cpp)
