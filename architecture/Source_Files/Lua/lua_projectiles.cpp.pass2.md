# Source_Files/Lua/lua_projectiles.cpp - Enhanced Analysis

## Architectural Role
This file is the primary **glue layer** between the GameWorld simulation and Lua scripting. It exposes live projectile state and provides mutation capabilities gated by a mutability interface. As part of the Lua subsystem, it depends directly on GameWorld (`projectiles.h`, `monsters.h`, `player.h`) and Files (for definition data), serving as the public API for script-driven projectile inspection and manipulation.

## Key Cross-References

### Incoming (who depends on this)
- **Lua scripting layer** via `Lua_Projectiles_register()`: All Lua scripts access projectiles through the global `Projectiles` container or through owner/target references from monsters
- **Lua compatibility layer**: Legacy functions (e.g., `get_projectile_owner()`, `set_projectile_position()`) defined in embedded Lua script delegate to property accessors
- **GameWorld subsystem indirectly**: Projectile state is mutated via `Lua_Projectile_Set_*` methods, affecting simulation physics (gravity, damage scaling, targeting)

### Outgoing (what this file depends on)
- **GameWorld/projectiles.h**: Calls `remove_projectile()`, `new_projectile()`, `get_projectile_data()` for projectile lifecycle
- **GameWorld/map.h**: Calls `remove_object_from_polygon_object_list()` and `add_object_to_polygon_object_list()` when repositioning projectiles across spatial partitions
- **GameWorld/monsters.h, player.h**: Cross-references owner/target indices; `Lua_Monster::Push()` and `Lua_Player::Is()` for ownership/targeting conversions
- **GameWorld/map.h**: Accesses `object_data` (position, facing) and `projectile_data` (gravity, elevation, damage_scale) structures
- **Files/projectile_definitions.h**: Reads static projectile type definitions via `get_projectile_definition()` for damage data exposure
- **Dynamic limits**: Calls `get_dynamic_limit(_dynamic_limit_projectiles)` for projectile array bounds

## Design Patterns & Rationale

**Property-based accessor model**: Rather than method-style Lua calls (`projectile:get_damage_scale()`), properties are exposed via table metamethods (e.g., `projectile.damage_scale`). This matches Marathon's scripting conventions and is simpler than method registration.

**Data representation bridging**: Fixed-point internal math (`FIXED_ONE`, `WORLD_ONE`) is converted to Lua doubles on read; Lua doubles are scaled back on write. The `AngleConvert` constant (360 / FULL_CIRCLE) normalizes internal angle units to degreesΓÇöa lossy conversion that scripts see as 0ΓÇô360┬░.

**Conditional API registration**: Mutable operations (delete, position, new) are only registered if `m.world_mutable()` is true. Sound operations require `m.sound_mutable()`. This allows read-only scripts in some contexts (e.g., HUD rendering, observation modes) while enabling full scripting in editable maps.

**Backward compatibility via Lua delegation**: Rather than maintaining parallel C++ wrappers for legacy APIs, the file embeds Lua code that wraps new property accessors. This minimizes C++ maintenance while keeping old script APIs working.

**Object indirection pattern**: Projectiles store `object_index` and access position/facing through a separate `object_data` structure. This shares geometric data across all world objects but decouples projectile-specific attributes (gravity, damage_scale, target_index) from shared object properties.

## Data Flow Through This File

**Read-only access** (primary use case):
1. Lua script accesses `Projectiles[i].property`
2. Metamethod triggers `Lua_Projectile_Get_*` function
3. Function dereferences projectile index ΓåÆ calls `get_projectile_data()` + `get_object_data()`
4. Raw fixed-point/angle values are scaled and returned as Lua numbers
5. Data flows: GameWorld ΓåÆ Lua (unidirectional, no engine state change)

**Write path** (gated by mutability):
1. Lua script assigns `Projectiles[i].property = value`
2. Metamethod triggers `Lua_Projectile_Set_*` function
3. Function validates input type, converts from Lua units to internal units, modifies `projectile_data` or `object_data` in-place
4. Special case: `Lua_Projectile_Position()` triggers spatial partition updates (removes/re-adds to polygon object list)
5. Data flows: Lua ΓåÆ GameWorld (synchronous, immediate engine effect)

**Projectile creation**:
1. `Lua_Projectiles_New_Projectile()` constructs `world_point3d` from script args, calls `::new_projectile()` with hardcoded velocity `(WORLD_ONE, 0, 0)`
2. Returns new projectile userdata; insertion is handled by engine

**Owner/Target resolution**:
1. Scripts can assign nil, numeric index, `Lua_Monster`, or `Lua_Player` userdata
2. Setter normalizes to a monster index; player references are resolved to playerΓåÆmonster_index
3. Stored as `short` indices with `NONE` sentinel

## Learning Notes

This file exemplifies **cross-subsystem glue in a mature engine**: the clean separation between GameWorld (C++ simulation) and Lua (scripting API) is maintained through careful type conversion and registration gating. Key lessons:

- **Fixed-point arithmetic bridging**: The `AngleConvert` constant and `/ FIXED_ONE` / `/ WORLD_ONE` scaling show how to expose low-level numeric formats to high-level languages without leaking implementation details.
- **Capability-driven API**: Registering methods conditionally (`m.world_mutable()`, `m.sound_mutable()`) is a lightweight alternative to runtime permission checks on every call.
- **Backward compatibility through delegation**: Embedding a small Lua script to wrap new APIs is cheaper (in code size) than maintaining C++ wrapper functions, and it's idiomatic for a Lua-heavy engine.
- **Spatial data structure awareness**: The position setter understands that moving an object requires updating spatial indices (`add_object_to_polygon_object_list`, `remove_object_from_polygon_object_list`), showing tight coupling with map subsystem details.
- **Index-based object references**: Unlike modern engines using handles or GUIDs, this uses raw array indices (owner_index, target_index, polygon) with bounds checking only at registration time (`Lua_Projectile_Valid`), a pattern typical of 1990s/2000s game code.

## Potential Issues

- **Unchecked pointer dereferencing**: `get_projectile_data()` and `get_object_data()` return raw pointers with no null check; invalid indices passed from Lua could crash the engine (though `Lua_Projectile_Valid` guards against most cases).
- **No validity checks on owner/target**: Setting `owner` or `target` to a monster index doesn't verify the monster is alive or valid; dangling references could cause crashes during simulation if that monster is deleted.
- **Implicit polygon list consistency**: Calling `position()` with an invalid polygon index could leave the spatial partition in an inconsistent state if validation fails mid-operation.
- **Angle normalization lossy**: Converting angles through `AngleConvert` (FULL_CIRCLE ΓåÆ 360┬░) and back involves rounding; scripts may see drift over repeated get/set cycles.
