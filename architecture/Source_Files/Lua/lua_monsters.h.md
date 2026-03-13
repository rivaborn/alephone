# Source_Files/Lua/lua_monsters.h

## File Purpose
Declares Lua bindings for monster management in the Aleph One game engine. Exposes individual monster instances, monster collections, and enumerations (monster types and actions) to Lua scripts via template-based C++ wrapper classes.

## Core Responsibilities
- Declare Lua-facing type aliases for monsters (single instances and containers)
- Define enumeration bindings for monster actions and types
- Provide a registration function to initialize Lua bindings with mutability constraints
- Bridge game engine monster data structures to Lua scripting API

## Key Types / Data Structures

| Name | Kind (struct/enum/class/typedef/interface/trait) | Purpose |
|------|-------|---------|
| `Lua_Monster` | typedef (template instantiation) | Single monster instance proxy; wraps `monster_data` for Lua access |
| `Lua_Monsters` | typedef (template instantiation) | Container of all monsters; iterable collection in Lua |
| `Lua_MonsterAction` | typedef (template instantiation) | Enum binding for monster action states (attacking, dying, idle, etc.) |
| `Lua_MonsterType` | typedef (template instantiation) | Enum binding for monster type identifiers (fighter, cyborg, enforcer, etc.) |

## Global / File-Static State

| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|---------|---------|
| `Lua_Monster_Name` | `extern char[]` | global | String name "monster" used as Lua metatable key |
| `Lua_Monsters_Name` | `extern char[]` | global | String name "Monsters" used as Lua table key |
| `Lua_MonsterAction_Name` | `extern char[]` | global | String name "monster_action" used for action enum |
| `Lua_MonsterType_Name` | `extern char[]` | global | String name "monster_type" used for type enum |

## Key Functions / Methods

### Lua_Monsters_register
- Signature: `int Lua_Monsters_register(lua_State *L, const LuaMutabilityInterface& m)`
- Purpose: Register all monster-related Lua bindings in the Lua state, applying mutability rules
- Inputs:
  - `L`: Lua state to register bindings into
  - `m`: Mutability interface controlling read/write access to monster properties
- Outputs/Return: Returns status code (0 for success, nonzero for error)
- Side effects (global state, I/O, alloc): Modifies Lua registry; creates metatables and method tables for monster classes; populates Lua global namespace
- Calls (direct calls visible in this file): Not visible (defined elsewhere, likely in .cpp file)
- Notes: Implementation details deferred to `.cpp` file; uses template registration mechanism from `lua_templates.h`

## Control Flow Notes
This is a header-only declaration file. Registration occurs at engine startup when `Lua_Monsters_register()` is called. Once registered, Lua scripts can:
1. Access monsters via the `Monsters` container (e.g., `Monsters[0]` to get first monster)
2. Query/modify individual monster properties via getter/setter methods registered in `L_Class` and `L_Enum`
3. Compare monster types and actions using enum equality operators
4. Iterate over active monsters using the container's iterator interface

## External Dependencies
- **Lua C API** (`lua.h`, `lauxlib.h`, `lualib.h`): Core scripting engine
- **cseries.h**: Game engine common utilities
- **map.h** (defined elsewhere): World/map data structures referenced by monsters
- **monsters.h** (defined elsewhere): Native `monster_data` structure and monster constants (types, actions, modes)
- **lua_templates.h** (defined elsewhere): Template classes (`L_Class`, `L_Enum`, `L_Container`) that provide the Lua binding infrastructure
- **LuaMutabilityInterface** (defined elsewhere): Struct controlling script-side mutation of monsters

**Not inferable from this file:**
- Monster property getters/setters (registered via `L_Class` mechanisms)
- Enum mnemonic strings (registered via `L_Enum` mechanisms)
- Whether the implementation is thread-safe or singleton-based
