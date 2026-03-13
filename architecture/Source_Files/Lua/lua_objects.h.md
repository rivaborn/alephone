# Source_Files/Lua/lua_objects.h

## File Purpose
Declares Lua scripting bindings for game world objects (effects, items, scenery, sounds). Provides template-based wrapper classes that expose game data structures to Lua, along with registration function for initializing these bindings at engine startup.

## Core Responsibilities
- Declare typedef aliases mapping C++ game objects to Lua-accessible wrappers
- Define containers (L_Container) and enumerations (L_Enum) for object collections
- Expose extern string identifiers used as Lua metatable names
- Declare the registration function to initialize all Lua object bindings
- Support lazy-evaluation for sound objects via L_LazyEnum template

## Key Types / Data Structures
All types are template specializations; none are defined here:

| Name | Kind (typedef template) | Purpose |
|------|-------------------------|---------|
| Lua_Effect | L_Class specialization | Single effect object accessible from Lua |
| Lua_Effects | L_Container specialization | Iterable collection of all effects |
| Lua_EffectType | L_Enum specialization | Effect type enumeration with mnemonic names |
| Lua_EffectTypes | L_EnumContainer specialization | Indexed lookup of effect type enums by number or name |
| Lua_Item | L_Class specialization | Single item object accessible from Lua |
| Lua_Items | L_Container specialization | Iterable collection of all items |
| Lua_ItemType | L_Enum specialization | Item type enumeration with mnemonic names |
| Lua_ItemTypes | L_EnumContainer specialization | Indexed lookup of item type enums by number or name |
| Lua_Scenery | L_Class specialization | Single scenery object accessible from Lua |
| Lua_Sceneries | L_Container specialization | Iterable collection of all scenery objects |
| Lua_Sound | L_LazyEnum specialization | Sound enumeration with deferred validation |
| Lua_Sounds | L_EnumContainer specialization | Indexed lookup of sound enums by number or name |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Lua_Effect_Name | extern char[] | global | Metatable identifier string for Effect class |
| Lua_Effects_Name | extern char[] | global | Metatable identifier string for Effects container |
| Lua_EffectType_Name | extern char[] | global | Metatable identifier string for EffectType enum |
| Lua_EffectTypes_Name | extern char[] | global | Metatable identifier string for EffectTypes container |
| Lua_Item_Name | extern char[] | global | Metatable identifier string for Item class |
| Lua_Items_Name | extern char[] | global | Metatable identifier string for Items container |
| Lua_ItemType_Name | extern char[] | global | Metatable identifier string for ItemType enum |
| Lua_ItemTypes_Name | extern char[] | global | Metatable identifier string for ItemTypes container |
| Lua_Scenery_Name | extern char[] | global | Metatable identifier string for Scenery class |
| Lua_Sceneries_Name | extern char[] | global | Metatable identifier string for Sceneries container |
| Lua_Sound_Name | extern char[] | global | Metatable identifier string for Sound enum |
| Lua_Sounds_Name | extern char[] | global | Metatable identifier string for Sounds container |

All are extern declarations; definitions are in corresponding .cpp file.

## Key Functions / Methods

### Lua_Objects_register
- **Signature:** `int Lua_Objects_register(lua_State *L, const LuaMutabilityInterface& m);`
- **Purpose:** Register all game object Lua bindings with the Lua state; called during engine initialization to expose Effects, Items, Scenery, and Sounds to scripts.
- **Inputs:** Lua state pointer; mutability interface defining read/write access control for scripts.
- **Outputs/Return:** Integer return code (likely LUA_OK on success).
- **Side effects:** Modifies Lua registry; adds metatables, methods, and global tables for each object type.
- **Calls:** Not visible from this header (implementation in lua_objects.cpp).
- **Notes:** Called once at startup; LuaMutabilityInterface likely controls whether scripts can modify vs. query objects.

## Control Flow Notes
This is a pure declaration header. Bindings are registered during engine initialization (before scripts run), creating Lua global tables (`Effects`, `Items`, `Scenery`, `Sounds`) and per-object methods via templates in `lua_templates.h`. Scripts then access game objects via these globals and iterate/index into collections.

## External Dependencies
- **Lua C API:** lua.h, lauxlib.h, lualib.h (Lua 5.2 scripting interface)
- **Game data:** items.h (item definitions), map.h (world geometry)
- **Lua infrastructure:** lua_templates.h (template wrapper classes L_Class, L_Container, L_Enum, L_LazyEnum, L_EnumContainer)
- **Cross-platform:** cseries.h (includes SDL2, standard types)
- **LuaMutabilityInterface:** defined elsewhere; controls script permissions
