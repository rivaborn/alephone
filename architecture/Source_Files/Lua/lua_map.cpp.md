# Source_Files/Lua/lua_map.cpp

## File Purpose
Implements Lua bindings for game map entities and world properties. Provides a C API bridge between Lua scripts and the engine's map/level data, including platforms, polygons, lines, lights, media, terminals, and annotations.

## Core Responsibilities
- Define and register Lua wrapper classes for ~25 map entity types using the Lua template system
- Implement property getters and setters for all map-accessible game world objects
- Manage read/write access control via `LuaMutabilityInterface` (conditionally register mutators)
- Validate indices and enforce type constraints for all Lua Γåö C++ conversions
- Provide compatibility layer for legacy map scripts via embedded Lua code
- Bridge between Lua's dynamic typing and C++'s type-safe world data structures

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `control_panel_definition` | struct | Control panel static properties (class, textures, sounds, item) |
| `line_data` | struct | Map line endpoints, sides, and geometry (imported) |
| `polygon_data` | struct | Polygon floor/ceiling, media, and platform references (imported) |
| `platform_data` | struct | Platform state, heights, speed, flags (imported) |
| `media_data` | struct | Liquid/media properties: height, flow, light (imported) |
| `light_data` | struct | Light state, intensity, animation function (imported) |
| `map_annotation` | struct | Editor annotation: polygon index, text, location (imported) |
| `L_Class`, `L_Enum`, `L_Container` | template | Template system for wrapping C++ objects in Lua (imported) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_*_Name[]` | char array | global (multiple) | Type identifiers for ~25 Lua class names |
| `lua_stash` | extern unordered_map<string, string> | global | Per-level persistent key-value store for scripts |
| `compatibility_script` | static const char* | static | Embedded Lua code providing legacy function wrappers |
| `EndpointList` | extern vector | global | Map line endpoints (read-only from Lua) |
| `LineList` | extern vector | global | Map lines (read-only from Lua) |
| `dynamic_world->platform_count` | global | Global access | Active platform count |
| `MediaList` | extern vector | global | Map media/liquid objects |
| `LightList` | extern vector | global | Map light sources |
| `MapAnnotationList` | extern vector | global | Editor annotations |

## Key Functions / Methods

### Lua_Map_register
- **Signature:** `int Lua_Map_register(lua_State *L, const LuaMutabilityInterface& m)`
- **Purpose:** Main registration function; sets up all map-related Lua classes and global objects.
- **Inputs:** Lua state, mutability interface (controls write access)
- **Outputs/Return:** 0 (success)
- **Side effects:** Registers ~25 Lua classes, sets up metatables, pushes Level global object
- **Calls:** `L_Class::Register()`, `L_Container::Register()`, `L_Enum::Register()`, `compatibility()`
- **Notes:** Conditionally registers setters based on `m.world_mutable()` and `m.sound_mutable()`. Called once at Lua interpreter init.

### Platform property functions (template instantiations)
- **Examples:** `Lua_Platform_Get_Active()`, `Lua_Platform_Set_Floor_Height()`, `Lua_Platform_Get_Static_Flag<_platform_is_locked>()`
- **Purpose:** Getter/setter pairs for platform state (active, height, speed, flags, type, tag)
- **Pattern:** Fetch platform via `get_platform_data()`, read/write field, validate range, push to Lua
- **Mutability:** Static flag getters always available; setters only if `world_mutable()`

### Polygon Floor/Ceiling property functions
- **Examples:** `Lua_Polygon_Floor_Get_Height()`, `Lua_Polygon_Floor_Set_Texture_Index()`
- **Purpose:** Manage floor/ceiling texture, height, light source, transfer mode
- **Side effects:** Height changes trigger `adjust_platform_endpoint_and_line_heights()` and media adjustment
- **Notes:** Texture indices validated against `MAXIMUM_SHAPES_PER_COLLECTION`

### Light property functions
- **Examples:** `Lua_Light_Get_Active()`, `Lua_Light_Set_Intensity()`
- **Purpose:** Manage light state, intensity, tag, preset type
- **Mutability:** Getters always available; setters only if `world_mutable()` or `sound_mutable()`

### Media property functions
- **Examples:** `Lua_Media_Set_High()`, `Lua_Media_Set_Type()`
- **Purpose:** Manage media height, flow direction/speed, light, type
- **Side effects:** Changes invoke `update_one_media()` to recalculate physics
- **Mutability:** Setters only if `world_mutable()`

### Annotation functions
- **Signature:** `int Lua_Annotations_New(lua_State *L)`
- **Purpose:** Create new map annotation with text, polygon, position
- **Inputs:** polygon (optional), text string, x, y (optional)
- **Outputs:** Pushes new Lua_Annotation object
- **Side effects:** Appends to `MapAnnotationList`, increments `dynamic_world->default_annotation_count`
- **Notes:** Limits to `INT16_MAX` annotations

### Fog property functions
- **Examples:** `Lua_Fog_Get_Depth()`, `Lua_Fog_Set_Mode()`
- **Purpose:** Manage OpenGL fog (above/below liquid): active, depth, color (RGB), mode, landscape mix
- **Calls:** `OGL_GetFogData()` to retrieve fog state
- **Mutability:** Getters always available; setters if `world_mutable() || fog_mutable()`

### compatibility()
- **Signature:** `static void compatibility(lua_State *L)`
- **Purpose:** Load legacy Lua compatibility wrapper functions for backwards-compatible map scripts
- **Side effects:** Executes embedded Lua code defining ~30 wrapper functions (e.g., `get_fog_depth()`, `set_platform_state()`)
- **Notes:** Embeds an entire Lua library as a static string; wraps new object-oriented API with old procedural function names

## Control Flow Notes
This file is **not frame-driven**; it defines a static registry of Lua bindings:
1. **Init phase:** `Lua_Map_register()` called once during Lua interpreter setup
2. **Runtime:** Lua scripts call getters/setters via the registered metatables; C functions execute on-demand
3. **Shutdown:** Implicit (Lua GC handles object cleanup)

No active loop or update function. Property access is synchronous: script calls ΓåÆ C getter/setter ΓåÆ world data read/write ΓåÆ return to Lua.

## External Dependencies
- **Engine headers:** `interface.h` (game state), `network.h` (game_info), `map.h` (world data), `lightsource.h` (light_data), `platforms.h`, `player.h`, `projectiles.h`, `OGL_Setup.h` (OpenGL fog), `SoundManager.h`
- **Lua API:** `lua.h`, `lauxlib.h`, `lualib.h` (standard Lua C API)
- **Template system:** `lua_templates.h` (L_Class, L_Enum, L_Container)
- **Other Lua modules:** `lua_monsters.h`, `lua_objects.h`, `lua_player.h` (related entity bindings)
- **Imported symbols (defined elsewhere):**
  - `get_platform_data()`, `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `get_media_data()`, `get_light_data()`
  - `EndpointList`, `LineList`, `MediaList`, `LightList`, `MapAnnotationList`
  - `dynamic_world`, `static_world` (global world state)
  - `set_platform_state()`, `adjust_platform_endpoint_and_line_heights()`, `update_one_media()`
  - `OGL_GetFogData()`, `OGL_Fog_AboveLiquid`, `OGL_Fog_BelowLiquid`
