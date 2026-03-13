# Source_Files/Lua/lua_monsters.cpp

## File Purpose
Implements Lua scripting bindings for monster systems in a game engine (appears to be Marathon/Aleph One). Exposes monster types, individual monsters, their properties, relationships, and actions to the Lua scripting environment.

## Core Responsibilities
- **Lua type registration** for monsters, monster types, monster classes, modes, and actions
- **Property access** (getters/setters) for monster attributes (health, position, facing, active state, etc.)
- **Relationship management** between monster types (enemies, friends, alliances)
- **Damage interactions** (immunities, weaknesses for specific damage types)
- **Instance operations** (movement, acceleration, targeting, sound playback, deletion)
- **Backward compatibility** layer providing old API through Lua wrapper functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_MonsterClass` | typedef (enum wrapper) | Typed wrapper for monster classification indices (flags like _class_fighter, _class_cyborg, _class_yeti) |
| `Lua_MonsterType_Enemies` | typedef (class wrapper) | Metatable-based table for querying/setting which monster types are hostile to each other |
| `Lua_MonsterType_Friends` | typedef (class wrapper) | Metatable-based table for querying/setting allied monster relationships |
| `Lua_MonsterType_Immunities` | typedef (class wrapper) | Damage type immunity table (bitfield: which damage types don't affect this monster type) |
| `Lua_MonsterType_Weaknesses` | typedef (class wrapper) | Damage type weakness table (bitfield: which damage types deal extra/only damage) |
| `Lua_MonsterType` | typedef (enum wrapper) | Typed wrapper for monster type indices (0 to NUMBER_OF_MONSTER_TYPES-1) |
| `Lua_MonsterMode` | typedef (enum wrapper) | Monster AI mode (locked, losing_lock, lost_lock, unlocked, running) |
| `Lua_MonsterAction` | typedef (enum wrapper) | Monster animation/action state (stationary, moving, attacking, dying, teleporting, etc.) |
| `Lua_Monster` | typedef (class wrapper) | Typed wrapper for individual monster instance indices |
| `Lua_Monsters` | typedef (container) | Global Monsters collection (callable, iterable, indexable by number or type) |
| `monster_pathfinding_data` | struct | Pathfinding context: holds monster definition, monster instance, and boundary-crossing flag |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AngleConvert` | const float | file-static | Conversion factor (360/FULL_CIRCLE) for angle unit conversions |
| `Lua_MonsterClass_Name[]` | char[] | file-static | String literal "monster_class" for Lua type registry key |
| `Lua_MonsterClasses_Name[]` | char[] | file-static | String literal "MonsterClasses" for Lua global |
| `Lua_MonsterType_Enemies_Name[]` | char[] | file-static | String literal "monster_type_enemies" |
| `Lua_MonsterType_Friends_Name[]` | char[] | file-static | String literal "monster_type_friends" |
| `Lua_MonsterType_Immunities_Name[]` | char[] | file-static | String literal "monster_type_immunities" |
| `Lua_MonsterType_Weaknesses_Name[]` | char[] | file-static | String literal "monster_type_weaknesses" |
| `Lua_MonsterType_Name[]` | char[] | file-static | String literal "monster_type" |
| `Lua_MonsterMode_Name[]` | char[] | file-static | String literal "monster_mode" |
| `Lua_MonsterModes_Name[]` | char[] | file-static | String literal "MonsterModes" |
| `Lua_MonsterAction_Name[]` | char[] | file-static | String literal "monster_action" |
| `Lua_MonsterActions_Name[]` | char[] | file-static | String literal "MonsterActions" |
| `Lua_MonsterTypes_Name[]` | char[] | file-static | String literal "MonsterTypes" |
| `Lua_Monster_Name[]` | char[] | file-static | String literal "monster" |
| `Lua_Monsters_Name[]` | char[] | file-static | String literal "Monsters" |
| `monster_placement_info` | object_frequency_definition* | extern | Pointer to placement data (initial/minimum/maximum counts, random spawn flags) |
| `Lua_MonsterType_Get[]` | luaL_Reg[] | file-static | Metatable for monster type getter methods (~45 properties) |
| `Lua_MonsterType_Set[]` | luaL_Reg[] | file-static | Metatable for monster type setter methods |
| `Lua_Monster_Get[]` | luaL_Reg[] | file-static | Metatable for monster instance getter methods (~20 properties) |
| `Lua_Monster_Get_Mutable[]` | luaL_Reg[] | file-static | Additional methods for world-mutable contexts (accelerate, attack, damage, delete, move_by_path, position) |
| `Lua_Monster_Get_Sound[]` | luaL_Reg[] | file-static | Additional methods for sound-enabled contexts (play_sound) |
| `Lua_Monster_Set[]` | luaL_Reg[] | file-static | Metatable for monster instance setter methods |
| `Lua_Monsters_Methods[]` | luaL_Reg[] | file-static | Methods for Monsters container (new) |
| `compatibility_script` | const char[] | file-static | Lua code string defining backward-compatible API wrappers |

## Key Functions / Methods

### Lua_MonsterClass_Valid
- Signature: `bool Lua_MonsterClass_Valid(int32 index)`
- Purpose: Validates that a monster class index is in the valid range and is a power-of-two (bitflag).
- Inputs: `index` ΓÇö class bitflag to validate
- Outputs/Return: `true` if 1 Γëñ index Γëñ _class_yeti and index is a power-of-two
- Side effects: None
- Calls: (bitwise operations only)
- Notes: Relies on monster classes being power-of-two bitmasks.

### Lua_MonsterType_Enemies_Get / Lua_MonsterType_Enemies_Set
- Signature: `int Lua_MonsterType_Enemies_Get(lua_State *L)` / `int Lua_MonsterType_Enemies_Set(lua_State *L)`
- Purpose: Read/write which monster classes are enemies to a given monster type.
- Inputs: L[1] = monster_type (index), L[2] = enemy_class, L[3] = bool (set only)
- Outputs/Return: 1 (boolean result on stack for Get); 0 (modifies definition in-place for Set)
- Side effects: `Lua_MonsterType_Enemies_Set` modifies global `monster_definition::enemies` bitfield
- Calls: `get_monster_definition_external()`
- Notes: Bitfield manipulation; uses bitwise OR to enable, AND NOT to disable.

### Lua_MonsterType_Get_* / Lua_MonsterType_Set_*
- Examples: `Lua_MonsterType_Get_Class`, `Lua_MonsterType_Get_Height`, `Lua_MonsterType_Get_Radius`, `Lua_MonsterType_Set_Class`, etc.
- Purpose: Accessor methods for read-only properties (class, collection, height, radius, vitality, impact effects, placement counts, flags) and mutable properties.
- Inputs: L[1] = monster_type index; L[2] = value (setters only)
- Outputs/Return: 1 (value pushed to Lua stack)
- Side effects: Setters modify external `monster_definition` or `monster_placement_info` arrays
- Calls: `get_monster_definition_external()`
- Notes: Many use templated flag accessors (`Lua_MonsterType_Get_Flag<flag>`) for boolean bit-tests; dimension conversions scale by WORLD_ONE.

### Lua_Monster_Accelerate
- Signature: `int Lua_Monster_Accelerate(lua_State *L)`
- Purpose: Apply velocity and vertical-velocity changes to a monster in world space.
- Inputs: L[1] = monster_index, L[2] = direction (degrees 0ΓÇô360), L[3] = velocity, L[4] = vertical_velocity
- Outputs/Return: 0 (no return value)
- Side effects: Calls `accelerate_monster()` with unit conversions
- Calls: `get_monster_data()`, `accelerate_monster()`
- Notes: Converts angle from degrees to internal units using `AngleConvert`; converts velocities by WORLD_ONE.

### Lua_Monster_Attack
- Signature: `int Lua_Monster_Attack(lua_State *L)`
- Purpose: Set or clear a monster's current attack target.
- Inputs: L[1] = monster_index, L[2] = target (nil, number index, or monster instance)
- Outputs/Return: 0
- Side effects: Calls `change_monster_target()`
- Calls: `get_monster_data()`, `change_monster_target()`
- Notes: Nil sets target to -1 (none); accepts both raw indices and Lua_Monster instances.

### Lua_Monster_Damage
- Signature: `int Lua_Monster_Damage(lua_State *L)`
- Purpose: Inflict damage on a monster with optional damage type.
- Inputs: L[1] = monster_index, L[2] = damage_amount, L[3] = damage_type (optional)
- Outputs/Return: 0
- Side effects: Modifies monster vitality; triggers death animations if lethal
- Calls: `get_monster_data()`, `damage_monster()`
- Notes: Defaults to `_damage_fist` if no type specified; damage_definition struct fields are hardcoded (flags=0, scale=FIXED_ONE).

### Lua_Monster_Delete
- Signature: `int Lua_Monster_Delete(lua_State *L)`
- Purpose: Remove a monster from the world and invalidate all Lua references to it.
- Inputs: L[1] = monster_index
- Outputs/Return: 0 or luaL_error if monster is player
- Side effects: Sets action to dying_soft, calls `monster_died()`, removes object, marks slot free, invalidates Lua instance
- Calls: `get_monster_data()`, `monster_died()`, `remove_map_object()`, `object_was_just_destroyed()`, `L_Invalidate_Monster()`, `MARK_SLOT_AS_FREE()`
- Notes: Prevents aggressor monsters from re-targeting by setting dying_soft action; handles promotion/demotion flags.

### Lua_Monster_Move_By_Path
- Signature: `int Lua_Monster_Move_By_Path(lua_State *L)`
- Purpose: Command a monster to pathfind to a destination polygon.
- Inputs: L[1] = monster_index, L[2] = destination_polygon (number or Lua_Polygon instance)
- Outputs/Return: 0
- Side effects: Creates/deletes pathfinding path; may activate monster; sets path segment flags; calls advance_monster_path
- Calls: `get_monster_data()`, `activate_monster()`, `delete_path()`, `new_path()`, `advance_monster_path()`, `set_monster_action()`, `set_monster_mode()`
- Notes: Uses `monster_pathfinding_cost_function` callback; allows crossing zone boundaries; returns early if path creation fails.

### Lua_Monster_Position
- Signature: `int Lua_Monster_Position(lua_State *L)`
- Purpose: Teleport a monster to a new world location and polygon.
- Inputs: L[1] = monster_index, L[2ΓÇô4] = x, y, z (world coordinates), L[5] = polygon_index
- Outputs/Return: 0
- Side effects: Modifies object location; updates object polygon list if polygon changed
- Calls: `get_monster_data()`, `get_object_data()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
- Notes: Coordinates are scaled by WORLD_ONE for conversion.

### Lua_Monster_Get_* / Lua_Monster_Set_*
- Examples: `Lua_Monster_Get_Action`, `Lua_Monster_Get_Facing`, `Lua_Monster_Get_Active`, `Lua_Monster_Set_Visible`, `Lua_Monster_Set_Vitality`, etc.
- Purpose: Instance property accessors (action, mode, location, vitality, active state, facing angle, visibility, flag states).
- Inputs: L[1] = monster_index; L[2] = value (setters only)
- Outputs/Return: 1 (value on stack)
- Side effects: Setters call `activate_monster()`, `deactivate_monster()`, or modify object state
- Calls: `get_monster_data()`, `get_object_data()`, `activate_monster()`, `deactivate_monster()`
- Notes: Angle conversions use `AngleConvert`; dimensions scale by WORLD_ONE; flag setters may enforce pre-activation constraints.

### Lua_Monsters_New
- Signature: `int Lua_Monsters_New(lua_State *L)`
- Purpose: Spawn a new monster instance in the world.
- Inputs: L[1ΓÇô3] = x, y, z (world coords), L[4] = polygon_index, L[5] = monster_type
- Outputs/Return: 1 (Lua_Monster instance pushed to stack) or 0 if creation fails
- Side effects: Allocates monster data, registers in object system
- Calls: `new_monster()`
- Notes: Constructs `object_location` struct with zero yaw/pitch; coordinates scaled by WORLD_ONE.

### Lua_Monsters_register
- Signature: `int Lua_Monsters_register(lua_State *L, const LuaMutabilityInterface& m)`
- Purpose: Register all monster-related Lua types and methods with a Lua state; conditionally enable mutable operations.
- Inputs: `L` = Lua state, `m` = mutability interface (world_mutable, sound_mutable)
- Outputs/Return: 0
- Side effects: Registers ~15 Lua types, populates Lua globals (MonsterClasses, MonsterTypes, MonsterModes, MonsterActions, Monsters), loads compatibility script
- Calls: `Lua_MonsterClass::Register()`, `Lua_MonsterType::Register()`, `Lua_Monster::Register()`, `Lua_Monsters::Register()`, `compatibility()`, many template methods
- Notes: World mutability gates add/delete/modify operations; sound mutability gates sound playback; compatibility layer loaded at end.

### compatibility (function) / compatibility_script (data)
- Signature: `static void compatibility(lua_State *L)` and companion `compatibility_script` Lua code
- Purpose: Provide backward-compatible API wrapper functions that map old monster function names to new object-oriented Lua API.
- Inputs: Lua code defining functions like `activate_monster()`, `get_monster_facing()`, `set_monster_position()`, etc.
- Outputs/Return: Functions registered in Lua global namespace
- Side effects: Loads and executes Lua code; no C-side state changes
- Calls: `luaL_loadbuffer()`, `lua_pcall()`
- Notes: ~30 wrapper functions; some perform unit conversions (e.g., facing 512/360 for old API).

## Control Flow Notes
1. **Initialization**: `Lua_Monsters_register()` is called once at engine startup to register all types and enable scripting.
2. **Property Access**: Lua code accesses monster properties via `__index`/`__newindex` metamethods, which dispatch to getter/setter C functions in metatable arrays.
3. **Instance Operations**: Commands like `monster:attack(target)`, `monster:damage(amt)`, `monster:position(x,y,z,poly)` are routed through `Lua_Monster_Get_Mutable[]` and `Lua_Monster_Set[]`.
4. **Containers**: `Monsters` and `MonsterTypes` are iterable collections with length and random-access by number.
5. **Backward Compat**: Old-style `activate_monster(idx)`, `get_monster_facing(idx)` calls are converted to new-style `Monsters[idx].active = true`, `Monsters[idx].facing`, etc.

## External Dependencies
- **`monsters.h`** ΓÇö `monster_data` struct, `monster_definition` struct, monster constants, activation/movement/damage functions
- **`player.h`** ΓÇö player data access, `monster_index_to_player_index()`
- **`flood_map.h`** ΓÇö pathfinding API (`new_path()`, `delete_path()`, `monster_pathfinding_cost_function`)
- **`lua_templates.h`** ΓÇö `L_Class`, `L_Enum`, `L_Container`, `L_EnumContainer` template classes for Lua binding
- **`lua_map.h`** ΓÇö `Lua_Polygon`, `Lua_Collection`, `Lua_DamageType` (cross-references for spatial and damage data)
- **`lua_objects.h`** ΓÇö `Lua_EffectType`, `Lua_Sound`, `Lua_ItemType` (cross-references for effects and items)
- **`lua_player.h`** ΓÇö `Lua_Player` (cross-reference for player instances)
- **Lua C API** ΓÇö `lua.h`, `lauxlib.h`, `lualib.h`
- **Game engine core** ΓÇö object management, world coordinate system, effect/sound playback
