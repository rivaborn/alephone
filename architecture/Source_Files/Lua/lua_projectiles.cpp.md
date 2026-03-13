# Source_Files/Lua/lua_projectiles.cpp

## File Purpose
Implements Lua C API bindings for projectile game objects, allowing game scripts to inspect and manipulate projectiles in the game world. Provides getters/setters for projectile properties, factory methods for creation, and a backward-compatibility layer for legacy script APIs.

## Core Responsibilities
- Register individual projectile accessors (position, orientation, ownership, targeting) with Lua
- Register the Projectiles container for indexed access to all active projectiles
- Provide projectile creation and deletion through Lua scripting
- Expose projectile type definitions and damage properties to scripts
- Convert between game engine units (fixed-point, angles) and Lua number formats
- Maintain backward-compatible Lua functions wrapping the new property-based API

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `projectile_data` | struct (external) | Engine representation of an active projectile; accessed via `get_projectile_data()` |
| `object_data` | struct (external) | Underlying object data (position, facing, shape, animation) shared by all world objects |
| `projectile_definition` | struct (external) | Static definition of a projectile type (damage, behavior); accessed via `get_projectile_definition()` |
| `damage_definition` | struct (external) | Damage type, base/random damage, and scaling for weapons/projectiles |
| `world_point3d` | struct (external) | 3D position in world coordinates; scaled by `WORLD_ONE` internally |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AngleConvert` | const float | static | Conversion factor: `360 / FULL_CIRCLE` for internal angle Γåö degree conversion |
| `Lua_Projectile_Name` | char[] | extern | String identifier `"projectile"` for Lua metatable |
| `Lua_Projectiles_Name` | char[] | extern | String identifier `"Projectiles"` for container global |
| `Lua_ProjectileType_Name` | char[] | extern | String identifier `"projectile_type"` for type enum |
| `Lua_ProjectileTypes_Name` | char[] | extern | String identifier `"ProjectileTypes"` for type container |
| `compatibility_script` | const char[] | static | Lua function definitions for backward-compatible API (wrapped property access) |

## Key Functions / Methods

### Lua_Projectile_Delete
- **Signature:** `int Lua_Projectile_Delete(lua_State* L)`
- **Purpose:** Remove a projectile from the game world
- **Inputs:** Lua stack: projectile userdata at index 1
- **Outputs/Return:** 0 (no return value to Lua)
- **Side effects:** Calls `remove_projectile()` on the projectile index; projectile data becomes invalid
- **Calls:** `remove_projectile()`
- **Notes:** Called as a method: `projectile:delete()`

### Lua_Projectile_Get_X / Lua_Projectile_Get_Y / Lua_Projectile_Get_Z
- **Signature:** `static int Lua_Projectile_Get_X(lua_State *L)` (similarly for Y, Z)
- **Purpose:** Retrieve the 3D position component of a projectile
- **Inputs:** Lua stack: projectile userdata at index 1
- **Outputs/Return:** Pushes double (world coordinate scaled down by `WORLD_ONE`)
- **Side effects:** None
- **Calls:** `get_projectile_data()`, `get_object_data()`
- **Notes:** Position is stored in the associated `object_data` structure

### Lua_Projectile_Get_Facing / Lua_Projectile_Get_Elevation
- **Signature:** `static int Lua_Projectile_Get_Facing(lua_State *L)`, `static int Lua_Projectile_Get_Elevation(lua_State *L)`
- **Purpose:** Retrieve orientation: facing (yaw) in degrees, elevation (pitch) in degrees
- **Inputs:** Lua stack: projectile userdata at index 1
- **Outputs/Return:** Pushes double (degrees, normalized 0ΓÇô360 or ΓêÆ180ΓÇô180)
- **Side effects:** None
- **Calls:** `get_projectile_data()`, `get_object_data()`
- **Notes:** Facing is in object; elevation is in projectile_data. Elevation < 0 is adjusted by adding 360

### Lua_Projectile_Get_Owner / Lua_Projectile_Get_Target
- **Signature:** `static int Lua_Projectile_Get_Owner(lua_State *L)`, `static int Lua_Projectile_Get_Target(lua_State *L)`
- **Purpose:** Retrieve the monster/player who owns or is targeted by the projectile
- **Inputs:** Lua stack: projectile userdata at index 1
- **Outputs/Return:** Pushes Lua_Monster userdata (or nil if no owner/target)
- **Side effects:** None
- **Calls:** `get_projectile_data()`, `Lua_Monster::Push()`
- **Notes:** Stores indices; NONE indicates no owner/target

### Lua_Projectile_Get_Damage_Scale / Lua_Projectile_Get_Gravity
- **Signature:** `static int Lua_Projectile_Get_Damage_Scale(lua_State *L)`, `static int Lua_Projectile_Get_Gravity(lua_State *L)`
- **Purpose:** Retrieve projectile-specific damage multiplier and gravity/vertical acceleration
- **Inputs:** Lua stack: projectile userdata at index 1
- **Outputs/Return:** Pushes double (damage_scale / FIXED_ONE or gravity / WORLD_ONE)
- **Side effects:** None
- **Calls:** `get_projectile_data()`

### Lua_Projectile_Set_* (Facing, Elevation, Owner, Target, Damage_Scale, Gravity)
- **Signature:** `static int Lua_Projectile_Set_Facing(lua_State *L)`, etc.
- **Purpose:** Modify projectile properties
- **Inputs:** Lua stack: projectile at index 1, new value (number, monster, or nil) at index 2
- **Outputs/Return:** 0 (no Lua return)
- **Side effects:** Modifies `projectile_data` or `object_data` in-place
- **Calls:** `get_projectile_data()`, `get_object_data()`, `get_player_data()` (for owner/target)
- **Notes:** Owner/target accept nil, numeric index, monster userdata, or player userdata; setters validate argument types

### Lua_Projectile_Position
- **Signature:** `int Lua_Projectile_Position(lua_State *L)`
- **Purpose:** Atomically set projectile position and polygon, removing from old polygon list if necessary
- **Inputs:** Lua stack: projectile (index 1), x/y/z floats (2ΓÇô4), polygon index or userdata (5)
- **Outputs/Return:** 0
- **Side effects:** Updates location and polygon in object_data; calls `remove_object_from_polygon_object_list()` and `add_object_to_polygon_object_list()` if polygon changes
- **Calls:** `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
- **Notes:** Validates polygon index; handles both numeric and Lua_Polygon userdata arguments

### Lua_Projectile_Play_Sound
- **Signature:** `int Lua_Projectile_Play_Sound(lua_State *L)`
- **Purpose:** Play a sound effect at the projectile's current location
- **Inputs:** Lua stack: projectile (index 1), sound type (index 2)
- **Outputs/Return:** 0
- **Side effects:** Triggers audio playback at projectile location
- **Calls:** `get_projectile_data()`, `Lua_Sound::ToIndex()`, `play_object_sound()`

### Lua_Projectiles_New_Projectile
- **Signature:** `int Lua_Projectiles_New_Projectile(lua_State *L)`
- **Purpose:** Create and insert a new projectile in the game world
- **Inputs:** Lua stack: x/y/z floats (1ΓÇô3), polygon index or userdata (4), projectile type (5)
- **Outputs/Return:** Pushes new projectile userdata; returns 1
- **Side effects:** Calls `::new_projectile()` to allocate and initialize projectile; inserts into world
- **Calls:** `Lua_Polygon::Valid()`, `Lua_Polygon::Index()`, `Lua_ProjectileType::ToIndex()`, `::new_projectile()`
- **Notes:** Initial velocity vector is hardcoded to `(WORLD_ONE, 0, 0)`; owner/target set to NONE

### Lua_Projectile_Valid
- **Signature:** `bool Lua_Projectile_Valid(int32 index)`
- **Purpose:** Check if a projectile index refers to an active projectile
- **Inputs:** Projectile index (0ΓÇôMAXIMUM_PROJECTILES_PER_MAP)
- **Outputs/Return:** true if slot is in use, false otherwise
- **Side effects:** None
- **Calls:** `GetMemberWithBounds()`, `SLOT_IS_USED()` macro
- **Notes:** Bounds-checked; used by Lua class framework

### Lua_Projectiles_register
- **Signature:** `int Lua_Projectiles_register(lua_State *L, const LuaMutabilityInterface& m)`
- **Purpose:** Initialize all projectile Lua bindings at engine startup
- **Inputs:** Lua state, mutability flags (whether scripts can modify world/sound)
- **Outputs/Return:** 0
- **Side effects:** Registers metatables, methods, and container; conditionally registers mutable operations (delete, position, new) based on mutability flags; executes compatibility Lua code
- **Calls:** `Lua_Projectile::Register()`, `Lua_Projectile::RegisterAdditional()`, `Lua_Projectile::Valid` assignment, container/type registration, `luaL_loadbuffer()`, `lua_pcall()`
- **Notes:** Read-only access is always enabled; mutable operations (delete, position, new) and sound operations depend on `m.world_mutable()` and `m.sound_mutable()` flags

## Control Flow Notes

- **Initialization:** Called via `Lua_Projectiles_register()` during engine init; registers all Lua-accessible projectile APIs
- **Runtime:** Lua scripts access projectiles via the global `Projectiles` container (array-like) or through monster/item references
- **Getters:** Retrieve live data from `projectile_data` and `object_data` on each access
- **Setters:** Modify engine state directly; no deferred updates
- **Compatibility:** Legacy `get_projectile_*` and `set_projectile_*` functions are defined in Lua code and delegate to property access
- **Bounds:** Projectile creation is limited by `MAXIMUM_PROJECTILES_PER_MAP` (from dynamic limits); iteration skips invalid slots

## External Dependencies

- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h`
- **Game World:** `map.h` (world coordinates, polygons, objects), `monsters.h`, `player.h`, `projectiles.h` (core data)
- **Templates:** `lua_templates.h` (L_Class, L_Enum, L_Container, L_EnumContainer, L_TableFunction)
- **Limits:** `dynamic_limits.h` (MAXIMUM_PROJECTILES_PER_MAP via `get_dynamic_limit()`)
- **Utilities:** `projectile_definitions.h` (DONT_REPEAT_DEFINITIONS flag for static data)
- **Defined Elsewhere:**
  - `remove_projectile()`, `new_projectile()`, `get_projectile_data()`, `get_projectile_definition()`
  - `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
  - `get_object_data()`, `get_player_data()`, `play_object_sound()`
  - Lua wrapper types: `Lua_Monster`, `Lua_Player`, `Lua_ProjectileType`, `Lua_Polygon`, `Lua_Sound`, `Lua_DamageType`
