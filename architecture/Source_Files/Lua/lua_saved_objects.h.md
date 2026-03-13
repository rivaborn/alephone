# Source_Files/Lua/lua_saved_objects.h

## File Purpose
Declares Lua bindings for game map objects (goals, items, monsters, player starts, sounds) by typedef'ing template-based wrapper classes. Provides the registration entry point to expose these map structures to Lua scripts.

## Core Responsibilities
- Export extern char arrays defining Lua class/container names for map object types
- Define typedef wrappers combining `L_Class` and `L_Container` templates for six object types
- Declare the registration function that sets up all Lua bindings in the Lua state
- Act as a header-only bridge between C++ game objects and Lua

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Lua_Goal | typedef (L_Class template) | Single goal object bound to Lua |
| Lua_Goals | typedef (L_Container template) | Container of all goals, iterable in Lua |
| Lua_ItemStart | typedef (L_Class template) | Single item spawn location bound to Lua |
| Lua_ItemStarts | typedef (L_Container template) | Container of all item starts |
| Lua_MonsterStart | typedef (L_Class template) | Single monster spawn location bound to Lua |
| Lua_MonsterStarts | typedef (L_Container template) | Container of all monster starts |
| Lua_PlayerStart | typedef (L_Class template) | Single player spawn location bound to Lua |
| Lua_PlayerStarts | typedef (L_Container template) | Container of all player starts |
| Lua_SoundObject | typedef (L_Class template) | Single sound source bound to Lua |
| Lua_SoundObjects | typedef (L_Container template) | Container of all sound objects |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Lua_Goal_Name | extern char[] | global | String identifier "goal" for Lua metatable |
| Lua_Goals_Name | extern char[] | global | String identifier "Goals" for container metatable |
| Lua_ItemStart_Name | extern char[] | global | String identifier "item_start" for Lua metatable |
| Lua_ItemStarts_Name | extern char[] | global | String identifier "ItemStarts" for container metatable |
| Lua_MonsterStart_Name | extern char[] | global | String identifier "monster_start" for Lua metatable |
| Lua_MonsterStarts_Name | extern char[] | global | String identifier "MonsterStarts" for container metatable |
| Lua_PlayerStart_Name | extern char[] | global | String identifier "player_start" for Lua metatable |
| Lua_PlayerStarts_Name | extern char[] | global | String identifier "PlayerStarts" for container metatable |
| Lua_SoundObject_Name | extern char[] | global | String identifier "sound_object" for Lua metatable |
| Lua_SoundObjects_Name | extern char[] | global | String identifier "SoundObjects" for container metatable |

## Key Functions / Methods

### Lua_Saved_Objects_register
- **Signature:** `int Lua_Saved_Objects_register(lua_State* L, const LuaMutabilityInterface& m)`
- **Purpose:** Register all saved object types (goals, items, monsters, players, sounds) as Lua userdata classes and containers in the given Lua state
- **Inputs:** Lua state pointer; mutability interface (likely controls read/write permissions for objects)
- **Outputs/Return:** Integer status code (typical Lua convention: 0 = success or count of items pushed)
- **Side effects:** Modifies Lua registry; registers metatables, container methods, and get/set handlers
- **Calls:** Not visible in this file (implementation is elsewhere, likely in `lua_saved_objects.cpp`)
- **Notes:** The actual implementation of this function is not provided here; this file only declares it

## Control Flow Notes
This is a header-only declarations file with no control flow. It is included by Lua binding implementation files and by code that needs to invoke `Lua_Saved_Objects_register()` at engine initialization time (during Lua state setup).

## External Dependencies
- **Includes:**
  - `cseries.h` ΓÇö common C series utilities (macros, platform abstractions)
  - `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö Lua 5.2 C API headers
  - `map.h` ΓÇö game map structure definitions (these objects are map-related)
  - `lua_templates.h` ΓÇö template implementations for `L_Class` and `L_Container` wrapper classes
- **Defined Elsewhere:**
  - `LuaMutabilityInterface` ΓÇö type passed to registration function; defines mutability constraints (not inferable from this file)
  - Implementations of all the `L_Class` and `L_Container` member functions (in `lua_templates.h`)
