# Source_Files/Lua/lua_map.h

## File Purpose
This header declares Lua bindings for the game engine's map system, enabling Lua scripts to interact with map geometry and environmental properties. It defines type-safe wrapper classes around core game structures (polygons, lines, platforms, lights, media, etc.) via template instantiation. The file serves as the primary interface layer between Lua scripts and the native C++ game world.

## Core Responsibilities
- Declare Lua class wrappers for all major map entity types (geometric and semantic)
- Define enum containers for mutable map properties (damage types, fog modes, transfer modes, etc.)
- Expose map geometry collections as iterable Lua tables
- Provide the single registration entry point for map module initialization
- Support read-only access to map state via Lua-friendly interfaces

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Polygon` | L_Class wrapper | Polygon geometry (floor/ceiling/walls) |
| `Lua_Polygons` | L_Container wrapper | Indexed collection of all polygons |
| `Lua_Line` | L_Class wrapper | Line segment (boundary between areas) |
| `Lua_Lines` | L_Container wrapper | Collection of all lines |
| `Lua_Side` | L_Class wrapper | Wall surface with texture and flags |
| `Lua_Sides` | L_Container wrapper | Collection of all sides |
| `Lua_Platform` | L_Class wrapper | Moving platform with animation state |
| `Lua_Platforms` | L_Container wrapper | Collection of platforms |
| `Lua_Light` | L_Class wrapper | Dynamic light source |
| `Lua_Lights` | L_Container wrapper | Collection of lights |
| `Lua_Terminal` | L_Class wrapper | Interactive terminal/computer |
| `Lua_Terminals` | L_Container wrapper | Collection of terminals |
| `Lua_Tag` | L_Class wrapper | Trigger tag identifier |
| `Lua_Tags` | L_Container wrapper | Collection of tags |
| `Lua_Media` | L_Class wrapper | Liquid/media surface |
| `Lua_Medias` | L_Container wrapper | Collection of media |
| `Lua_DamageType` | L_Enum wrapper | Damage type mnemonic (explosion, projectile, lava, etc.) |
| `Lua_FogMode` | L_Enum wrapper | Fog rendering mode |
| `Lua_TransferMode` | L_Enum wrapper | Texture transfer/blending mode |
| `Lua_PlatformType` | L_Enum wrapper | Platform animation behavior |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Polygon_Name` | extern char[] | extern | Registry key for polygon metatable ("polygon") |
| `Lua_Polygons_Name` | extern char[] | extern | Registry key for polygon container ("Polygons") |
| `Lua_Line_Name` | extern char[] | extern | Registry key for line ("line") |
| `Lua_Side_Name` | extern char[] | extern | Registry key for side ("side") |
| `Lua_Platform_Name` | extern char[] | extern | Registry key for platform ("platform") |
| `Lua_Light_Name` | extern char[] | extern | Registry key for light ("light") |
| `Lua_Terminal_Name` | extern char[] | extern | Registry key for terminal ("terminal") |
| `Lua_DamageType_Name` | extern char[] | extern | Registry key for damage type enum |
| (Similar for all ~40 other type names) | extern char[] | extern | Unique identifiers for Lua type registration |

## Key Functions / Methods

### `Lua_Map_register`
- **Signature**: `int Lua_Map_register (lua_State *L, const LuaMutabilityInterface& m);`
- **Purpose**: Registers the entire Lua map module, initializing all map entity types and collections as accessible Lua objects.
- **Inputs**:
  - `lua_State *L`: The Lua interpreter instance
  - `const LuaMutabilityInterface& m`: Permission interface controlling whether scripts can modify map state
- **Outputs/Return**: `int` - Lua status code (success/failure)
- **Side effects**: Registers metatables and container objects globally in Lua; makes all map structures accessible from scripts
- **Calls**: Indirectly calls template instantiation code for all `L_Class<...>::Register()` and `L_Container<...>::Register()` (defined in lua_templates.h)
- **Notes**: Single entry point for map module initialization; typically called once during engine startup. The `LuaMutabilityInterface` determines whether Lua scripts can write to map structures (editing during play vs. read-only access).

## Control Flow Notes

This header defines no control flowΓÇöit is purely declarative. The actual initialization occurs during engine startup when `Lua_Map_register()` is called (likely in a script initialization function). Once registered:
- **Runtime**: Lua scripts call methods on Lua_Polygon, iterate Lua_Polygons containers, query light states, etc.
- **Type safety**: Each template instantiation compiles to a distinct set of metamethods and accessor functions
- **Container iteration**: Collections (Lua_Polygons, Lua_Lines, etc.) support `for...in` loops in Lua via `__call` and `__len` metamethods

## External Dependencies

**Notable includes**:
- `cseries.h` ΓÇô Cross-platform utilities and SDL2 integration
- `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Lua C API (wrapped in `extern "C"` block)
- `map.h` ΓÇô Defines `polygon_data`, `line_data`, `side_data`, `object_data`, etc.
- `lightsource.h` ΓÇô Defines `light_data`, `static_light_data`
- `lua_templates.h` ΓÇô Provides `L_Class<>`, `L_Enum<>`, `L_Container<>`, `L_EnumContainer<>` templates for Lua binding metaprogramming

**External symbols (defined elsewhere)**:
- `lua_State`, Lua C API functions ΓÇô Lua interpreter
- `L_Class<N,T>::Register()`, `L_Enum<...>::Register()` ΓÇô Template methods from lua_templates.h
- `polygon_data`, `line_data`, `side_data`, `light_data` ΓÇô Game structures from map.h and lightsource.h
- `LuaMutabilityInterface` ΓÇô Permission control interface (definition not provided in bundled headers)
