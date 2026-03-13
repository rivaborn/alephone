# Source_Files/Lua/lua_saved_objects.cpp

## File Purpose
Implements Lua bindings for read-only access to saved map objects (goals, item/monster/player spawn points, sound emitters) defined in map files. Allows Lua scripts to query their positions, orientations, types, and flags without modifying them.

## Core Responsibilities
- Expose five map object types (Goals, ItemStarts, MonsterStarts, PlayerStarts, SoundObjects) as Lua classes
- Provide read-only getter methods for each object type (position, facing, polygon, type-specific metadata)
- Register Lua containers that allow iteration and indexing of all objects of each type
- Convert internal engine units (facing angles, world coordinates) to Lua-friendly formats
- Validate object indices and handle invalid object access

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Goal` | typedef (L_Class) | Lua binding for goal markers in map |
| `Lua_Goals` | typedef (L_Container) | Container of all goal objects |
| `Lua_ItemStart` | typedef (L_Class) | Lua binding for item spawn points |
| `Lua_ItemStarts` | typedef (L_Container) | Container of all item spawn points |
| `Lua_MonsterStart` | typedef (L_Class) | Lua binding for monster spawn points |
| `Lua_MonsterStarts` | typedef (L_Container) | Container of all monster spawn points |
| `Lua_PlayerStart` | typedef (L_Class) | Lua binding for player spawn points |
| `Lua_PlayerStarts` | typedef (L_Container) | Container of all player spawn points |
| `Lua_SoundObject` | typedef (L_Class) | Lua binding for ambient sound sources |
| `Lua_SoundObjects` | typedef (L_Container) | Container of all sound objects |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AngleConvert` | `const float` | static | Conversion factor from engine angle units to degrees (360/FULL_CIRCLE) |
| `Lua_Goal_Name`, `Lua_Goals_Name`, etc. | `char[]` | extern | Type name strings for Lua class registration (e.g., "goal", "Goals") |
| `saved_objects` | `map_object*` | extern | Array of all saved map objects; defined elsewhere |
| `dynamic_world` | pointer (implicit) | extern | World state containing initial object count; defined elsewhere |

## Key Functions / Methods

### get_map_object
- **Signature:** `map_object* get_map_object(short index)`
- **Purpose:** Retrieve a map object by its array index via pointer arithmetic.
- **Inputs:** `index` ΓÇô object index
- **Outputs/Return:** Pointer to the map object at that index
- **Side effects:** None (read-only)
- **Calls:** None
- **Notes:** Assumes `index` is valid; no bounds checking.

### get_saved_object_facing, get_saved_object_flag, get_saved_object_polygon, get_saved_object_x/y/z
- **Signature:** Template functions; e.g., `int get_saved_object_facing(lua_State* L)`
- **Purpose:** Lua getter methods for object properties (facing angle, flags, polygon index, world coordinates).
- **Inputs:** `L` ΓÇô Lua state; object index retrieved via `T::Index(L, 1)`
- **Outputs/Return:** 1 (pushes result to Lua stack)
- **Side effects:** Modifies Lua stack (pushes number, boolean, or object reference)
- **Calls:** `get_map_object()`, `lua_pushnumber()`, `lua_pushboolean()`, `Lua_Polygon::Push()`
- **Notes:** Coordinates are divided by `WORLD_ONE` to convert engine units. Angles multiplied by `AngleConvert` to convert to degrees.

### saved_objects_length
- **Signature:** `int saved_objects_length()`
- **Purpose:** Return the total count of saved objects in the current map.
- **Inputs:** None
- **Outputs/Return:** Object count from `dynamic_world->initial_objects_count`
- **Side effects:** None
- **Calls:** None (accesses global)
- **Notes:** Used as length provider for Lua container iteration.

### saved_object_valid
- **Signature:** `template<short T> bool saved_object_valid(short index)`
- **Purpose:** Validate that an index refers to a valid object of a specific type `T`.
- **Inputs:** `index` ΓÇô object index; `T` ΓÇô template parameter for expected object type
- **Outputs/Return:** `true` if index is in range and object type matches
- **Side effects:** None
- **Calls:** `get_map_object()`
- **Notes:** Used as the validation callback for Lua object classes.

### Lua_Goal_Get_ID, Lua_ItemStart_Get_Type, Lua_MonsterStart_Get_Type, Lua_PlayerStart_Get_Team
- **Signature:** `static int Lua_*_Get_*(lua_State* L)`
- **Purpose:** Type-specific getters that extract an ID or enum value from an object's index field and push a corresponding Lua type.
- **Inputs:** `L` ΓÇô Lua state; object index from `T::Index(L, 1)`
- **Outputs/Return:** 1 (pushes Lua enum value or nil)
- **Side effects:** Modifies Lua stack
- **Calls:** `get_map_object()`, `Lua_ItemType::Push()`, `Lua_MonsterType::Push()`, `Lua_PlayerColor::Push()`, `Lua_AmbientSound::Push()`, `Lua_Light::Push()`
- **Notes:** For `Lua_SoundObject_Get_Light` and `Lua_SoundObject_Get_Volume`, the `facing` field is repurposed to store light index (if negative) or volume (if non-negative).

### Lua_Saved_Objects_register
- **Signature:** `int Lua_Saved_Objects_register(lua_State* L, const LuaMutabilityInterface& m)`
- **Purpose:** Register all saved object types and their Lua classes with the Lua state during initialization.
- **Inputs:** `L` ΓÇô Lua state; `m` ΓÇô mutability interface (unused here)
- **Outputs/Return:** 0 (standard Lua convention)
- **Side effects:** Modifies Lua state; registers metatables and global containers
- **Calls:** `Lua_Goal::Register()`, `Lua_Goals::Register()`, and similar for each object type
- **Notes:** Sets the `Valid` callback and `Length` callback for each container class. Entry point called by Lua initialization system.

## Control Flow Notes
This is a **registration-phase** module, not a runtime loop. `Lua_Saved_Objects_register()` is invoked once during engine startup or map loading to expose map objects to the Lua scripting environment. After registration, Lua code can iterate over containers (e.g., `for goal in Goals() do ... end`) and access properties of individual objects. All read access is safe; no mutation is possible.

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Lua state and stack manipulation
- **Game Engine Core:** `map.h` ΓÇô `map_object` struct and polygon definitions; `monsters.h`, `items.h` ΓÇô type enumerations
- **Lua Template System:** `lua_templates.h` ΓÇô `L_Class`, `L_Container`, `L_Enum` metaprogramming
- **Sound Manager:** `SoundManagerEnums.h` ΓÇô sound constants (e.g., `MAXIMUM_SOUND_VOLUME`)
- **Included Files:** `lua_monsters.h`, `lua_objects.h`, `lua_player.h`, `lua_map.h` ΓÇô Lua bindings for related subsystems; `lua_saved_objects.h` ΓÇô declarations
