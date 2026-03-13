# Source_Files/Lua/lua_objects.cpp

## File Purpose
Implements Lua bindings for game world objects (effects, items, scenery). Provides scripting interface to manipulate map objects, query their properties, and control their lifecycle through registered getter/setter methods and object containers.

## Core Responsibilities
- Register Lua bindings for object types (Effect, Item, Scenery) and their enumerations with conditional mutability
- Provide getter/setter methods for object properties (position, facing, visibility, polygon, type)
- Implement object lifecycle operations (creation via constructors, deletion, teleportation)
- Expose object containers (Effects, Items, Sceneries) as Lua tables with iteration and indexing
- Support type lookups and enumeration (ItemType, SceneryType, EffectType with mnemonics)
- Maintain backward compatibility with deprecated Lua APIs

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Effect` | typedef (L_Class) | Effect object wrapper with index-based caching |
| `Lua_Effects` | typedef (L_Container) | Container enabling indexed/iterable access to effects |
| `Lua_Item` | typedef (L_Class) | Item object wrapper |
| `Lua_Items` | typedef (L_Container) | Container for items on map |
| `Lua_Scenery` | typedef (L_Class) | Scenery object wrapper |
| `Lua_Sceneries` | typedef (L_Container) | Container for scenery |
| `Lua_ItemType`, `Lua_SceneryType`, `Lua_EffectType` | typedef (L_Enum) | Type enumerations with mnemonic lookup |
| `Lua_ItemKind` | typedef (L_Enum) | Item classification enum |
| `luaL_Reg` arrays | struct array | Method registration tables (Get/Set/Mutable methods per type) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AngleConvert` | const float | file | Conversion factor: 360 / FULL_CIRCLE |
| `Lua_Effect_Name`, `Lua_Item_Name`, `Lua_Scenery_Name` | char[] | extern | Metatable names for Lua type identification |
| `Lua_*_Get[]`, `Lua_*_Set[]`, `Lua_*_Get_Mutable[]` | luaL_Reg[] | file-static | Method registration tables for getters/setters |
| `Lua_*_Methods[]` | luaL_Reg[] | file-static | Constructor tables (.new methods) |
| `compatibility_script` | static const char* | file-static | Lua code for deprecated API compatibility |

## Key Functions / Methods

### Template Functions (generic across object types T)

#### get_object_facing, get_object_x/y/z, get_object_polygon
- Signature: `template<class T> static int get_object_*(lua_State *L)`
- Purpose: Push object property (coordinate, angle, or polygon) to Lua stack
- Inputs: L (Lua state), parameter 1 (object index)
- Outputs/Return: 1 (pushes converted value; angles scaled by AngleConvert, coords scaled by 1/WORLD_ONE)
- Side effects: None
- Calls: `T::Index()`, `get_object_data()`, `lua_pushnumber()`

#### set_object_facing, set_object_visible
- Signature: `template<class T> static int set_object_*(lua_State *L)`
- Purpose: Update object property from Lua value
- Inputs: L, parameter 1 (object index), parameter 2 (new value)
- Outputs/Return: 0 (no return value); returns error on type mismatch
- Side effects: Modifies object_data in-place
- Calls: `T::Index()`, `get_object_data()`, `luaL_error()`
- Notes: Validates argument types (number, boolean)

#### lua_object_position
- Signature: `template<class T> int lua_object_position(lua_State *L)`
- Purpose: Set object location and polygon, updating spatial lists
- Inputs: L, params 2ΓÇô4 (x, y, z as world units), param 5 (polygon index as number or Lua_Polygon object)
- Outputs/Return: 0
- Side effects: Modifies object location; calls `remove_object_from_polygon_object_list()` and `add_object_to_polygon_object_list()` if polygon changes
- Calls: `T::Index()`, `get_object_data()`, `Lua_Polygon::Valid()`, `Lua_Polygon::Index()`, `luaL_error()`
- Notes: Handles both numeric and object polygon arguments; validates polygon index

### Effect-Specific Functions

#### Lua_Effects_New
- Signature: `int Lua_Effects_New(lua_State *L)`
- Purpose: Create a new effect at specified location
- Inputs: L, params 1ΓÇô3 (x, y, z), param 4 (polygon as number or Lua_Polygon), param 5 (effect_type)
- Outputs/Return: 1 (pushes Lua_Effect userdata); 0 if creation fails (returns NONE)
- Side effects: Allocates effect from dynamic pool; updates spatial lists
- Calls: `::new_effect()`, `Lua_Effect::Push()`, `Lua_EffectType::ToIndex()`

#### Lua_Effect_Delete, Lua_Effect_Position, Lua_Effect_Play_Sound
- Similar signatures to object equivalents but access effect via `effect_data*` and delegate to underlying object
- `Lua_Effect_Get_Facing/Polygon/X/Y/Z()` read from the effect's associated object_data

#### Lua_Effect_Valid
- Signature: `bool Lua_Effect_Valid(int32 index)`
- Purpose: Check if effect index is in-bounds and allocated
- Inputs: index
- Outputs/Return: true if 0 Γëñ index < MAXIMUM_EFFECTS_PER_MAP and slot is used
- Calls: `GetMemberWithBounds()`, `SLOT_IS_USED()`

### Item-Specific Functions

#### Lua_Items_New
- Signature: `int Lua_Items_New(lua_State *L)`
- Purpose: Create item with shape/placement setup
- Inputs: L, params 1ΓÇô3 (x, y, z), param 4 (polygon), param 5 (item_type)
- Outputs/Return: 1 (Lua_Item); 0 if failed
- Side effects: Allocates item object; calls `L_Get_Proper_Item_Accounting()` integration
- Calls: `::new_item()`, `Lua_Item::Push()`, `Lua_ItemType::ToIndex()`

#### Lua_Item_Delete
- Signature: `int Lua_Item_Delete(lua_State *L)`
- Purpose: Remove item and clean up associated teleport-in effects
- Inputs: L, parameter 1 (item index)
- Outputs/Return: 0
- Side effects: Removes object from map; iterates EffectList to find and remove `_effect_teleport_object_in` if item is invisible; notifies item accounting system
- Calls: `get_object_data()`, `OBJECT_IS_INVISIBLE()`, `remove_effect()`, `remove_map_object()`, `object_was_just_destroyed()`
- Notes: Handles teleporting items specially to clean up animation effects

#### Lua_ItemType_Get_* / Lua_ItemType_Set_*
- Signature: `static int Lua_ItemType_Get_Initial_Count(lua_State *L)` (and similar for max, min, random, etc.)
- Purpose: Query/modify item placement parameters (spawn counts, chances, flags)
- Inputs: L, parameter 1 (item_type index)
- Outputs/Return: 1 (pushes number/boolean); or error on type mismatch
- Side effects: Modifies `get_placement_info()[index]` or item definition fields
- Calls: `get_placement_info()`, `get_item_definition_external()`, `lua_pushnumber()`, `lua_pushboolean()`, `luaL_error()`

#### Lua_Item_Valid
- Signature: `bool Lua_Item_Valid(int32 index)`
- Purpose: Check if index is a valid in-map item
- Inputs: index
- Outputs/Return: true if object exists and owner is `_object_is_item`
- Calls: `GetMemberWithBounds()`, `SLOT_IS_USED()`, `GET_OBJECT_OWNER()`

### Scenery-Specific Functions

#### Lua_Sceneries_New
- Signature: `int Lua_Sceneries_New(lua_State *L)`
- Purpose: Create scenery object with random shape
- Inputs: L, params 1ΓÇô3 (x, y, z), param 4 (polygon), param 5 (scenery_type)
- Outputs/Return: 1 (Lua_Scenery); 0 if failed
- Side effects: Allocates scenery; calls `randomize_scenery_shape()` for visual variation
- Calls: `::new_scenery()`, `randomize_scenery_shape()`, `Lua_Scenery::Push()`

#### Lua_Scenery_Damage, Lua_Scenery_Get_Damaged/Solid
- Purpose: Damage scenery; query damaged/solid state
- Calls: `damage_scenery()`, `get_object_data()`, `GET_OBJECT_OWNER()`, `OBJECT_IS_SOLID()`

#### Lua_Scenery_Valid
- Signature: `static bool Lua_Scenery_Valid(int32 index)`
- Purpose: Check if index is valid scenery (excludes player monsters)
- Inputs: index
- Outputs/Return: true if object is scenery type or normal but not a player's body parts
- Side effects: None
- Calls: `GetMemberWithBounds()`, `SLOT_IS_USED()`, `GET_OBJECT_OWNER()`, `get_player_data()`, `get_monster_data()`, iterates player/monster table
- Notes: Filters out player legs/torso (parasitic objects)

### Registration

#### Lua_Objects_register
- Signature: `int Lua_Objects_register(lua_State *L, const LuaMutabilityInterface& m)`
- Purpose: Register all object types and methods with Lua VM on startup
- Inputs: L (Lua state), m (mutability flags for world/sound access)
- Outputs/Return: 0
- Side effects: Registers metatables, methods, containers, type enumerations; calls `compatibility()`
- Calls: `Lua_Effect::Register()`, `Lua_Item::Register()`, `Lua_Scenery::Register()`, enum type registrations, `Lua_Effects::Register()`, etc.
- Notes: Conditional registration of mutable (destructive) methods if `m.world_mutable()` true; sound methods registered if `m.sound_mutable()` or world mutable

#### compatibility
- Signature: `static void compatibility(lua_State *L)`
- Purpose: Load deprecated API shims as Lua script
- Inputs: L
- Outputs/Return: void
- Side effects: Executes Lua code defining backward-compatible functions
- Calls: `luaL_loadbuffer()`, `lua_pcall()`, `strlen()`

## Control Flow Notes
**Initialization phase only:** This file's code runs once when the Lua environment is set up. The `Lua_Objects_register()` function is called early in engine startup to expose game object APIs to Lua scripts. No participation in frame update or renderingΓÇöpurely registration and delegation.

Mutability gates control what operations are exposed:
- Read-only mode: only getters and type queries available
- World-mutable mode: allows object creation/deletion/modification
- Sound-mutable mode: allows sound playback on objects

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h`
- **Engine templates:** `lua_templates.h` (L_Class, L_Container, L_Enum infrastructure)
- **Map bindings:** `lua_map.h` (Lua_Polygon, Lua_Tags, etc.)
- **Game world modules:** `effects.h`, `items.h`, `monsters.h`, `scenery.h`, `player.h` (core data structures)
- **Sound system:** `SoundManager.h` 

**Defined elsewhere:**
- `remove_map_object()`, `get_object_data()`, `add_object_to_polygon_object_list()`, `remove_object_from_polygon_object_list()`
- `new_effect()`, `remove_effect()`, `get_effect_data()`, `teleport_object_in/out()`
- `new_item()`, `new_scenery()`, `damage_scenery()`, `deanimate_scenery()`, `randomize_scenery_shape()`
- `play_object_sound()`, `get_dynamic_limit()`, `get_placement_info()`, `get_item_definition_external()`
- Lua template method implementations and mnemonics tables (included from headers)
