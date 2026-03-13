# Source_Files/Lua/lua_ephemera.cpp

## File Purpose
Implements Lua bindings for ephemeraΓÇötransient visual objects in the game world. Provides Lua scripts with the ability to query, create, and modify ephemera properties (position, appearance, animation behavior).

## Core Responsibilities
- Register ephemera and ephemera quality as Lua classes/enums
- Expose ephemera property getters (position, facing, shape, collection, scale flags, animation flags)
- Expose ephemera property setters (for mutable contexts only)
- Implement ephemera creation (`new`) and deletion methods
- Implement position/polygon reassignment with spatial updates
- Gate write operations based on mutability context (world_mutable / ephemera_mutable)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_EphemeraQuality` | enum template | Maps ephemera quality levels (_off through _ultra) |
| `Lua_EphemeraQualities` | enum container | Collection of quality enum values |
| `Lua_Ephemera` | object class | Individual ephemera instance binding |
| `Lua_Ephemeras` | object container | Collection of all ephemera |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AngleConvert` | `const float` | static | Conversion factor: 360 / FULL_CIRCLE; converts internal angles to degrees |
| `Lua_Ephemera_Get` | `const luaL_Reg[]` | static | Registered getter functions (13 properties) |
| `Lua_Ephemera_Get_Mutable` | `const luaL_Reg[]` | static | Mutable methods table: delete, position |
| `Lua_Ephemera_Set` | `const luaL_Reg[]` | static | Registered setter functions (7 properties) |
| `Lua_Ephemera_Name` | `char[]` | global | Lua class name: "ephemera" |
| `Lua_Ephemeras_Name` | `char[]` | global | Lua container name: "Ephemera" |

## Key Functions / Methods

### set_shape, set_clut, set_collection
- Signature: `static uint16_t set_shape/clut/collection(uint16_t descriptor, uint16_t value)`
- Purpose: Bitwise utilities to pack/update shape descriptor fields
- Inputs: packed descriptor, new field value
- Outputs: updated descriptor
- Side effects: None
- Calls: None (pure bit manipulation)
- Notes: Descriptor layout uses bit offsets DESCRIPTOR_SHAPE_BITS and DESCRIPTOR_COLLECTION_BITS; no bounds checking (caller responsible)

### Lua_Ephemera_Position
- Signature: `static int Lua_Ephemera_Position(lua_State* L)`
- Purpose: Relocate ephemera to (x, y, z) and optionally change containing polygon
- Inputs: Lua args (ephemera, x, y, z, polygon_index_or_object)
- Outputs: 0 (no return)
- Side effects: Updates ephemera location; if polygon changes, removes from old polygon and adds to new
- Calls: `remove_ephemera_from_polygon`, `add_ephemera_to_polygon`
- Notes: Validates polygon validity; accepts both numeric index and Lua_Polygon object; scales world coords by WORLD_ONE

### Lua_Ephemeras_New
- Signature: `static int Lua_Ephemeras_New(lua_State* L)`
- Purpose: Create and register a new ephemera in the world
- Inputs: Lua args (x, y, z, polygon, collection, shape, facing)
- Outputs: 1 (pushes ephemera object) or 0 (allocation failed)
- Side effects: Allocates ephemera in world; scales coordinates and angles
- Calls: `new_ephemera`, `Lua_Ephemera::Push`
- Notes: Returns silently if `new_ephemera` returns NONE; validates polygon upfront

### Lua_Ephemera_Delete
- Signature: `static int Lua_Ephemera_Delete(lua_State* L)`
- Purpose: Remove ephemera from world
- Inputs: Lua arg (ephemera)
- Outputs: 0
- Side effects: Deallocates ephemera
- Calls: `remove_ephemera`
- Notes: No confirmation or validation

### Lua_Ephemera_Valid
- Signature: `static bool Lua_Ephemera_Valid(int32 index)`
- Purpose: Validity checker registered with Lua template system
- Inputs: ephemera index
- Outputs: true if index in bounds and slot in-use
- Side effects: None
- Calls: `get_dynamic_limit`, `get_ephemera_data`, `SLOT_IS_USED` macro
- Notes: Called by Lua template layer before dispatching object methods

### Lua_Ephemera_register
- Signature: `int Lua_Ephemera_register(lua_State* L, const LuaMutabilityInterface& m)`
- Purpose: Register all ephemera types, methods, and enums with Lua state
- Inputs: Lua state, mutability interface (gates read-only vs. read-write)
- Outputs: 0
- Side effects: Registers Lua_EphemeraQuality enum (5 values), Lua_Ephemera class, Lua_Ephemeras container; conditionally adds delete/position/new based on `m.world_mutable() || m.ephemera_mutable()`
- Calls: `Lua_EphemeraQuality::Register`, `Lua_EphemeraQualities::Register`, `Lua_Ephemera::Register`, `Lua_Ephemeras::Register`; binds `Lua_Ephemera_Valid` and length accessors
- Notes: Immutable contexts expose only getters; mutable contexts expose setters and creation

**Property getters (13):** clut_index, collection, end_when_animation_loops, enlarged, facing, polygon, rendered, shape_index, tiny, valid, x, y, z. All extract via `get_ephemera_data()`, apply macros for flag/field extraction, push result to Lua.

**Property setters (7):** clut_index, collection, end_when_animation_loops, enlarged, facing, shape_index, tiny. Validate input type, extract object, modify flags or call `set_ephemera_shape()` for descriptor changes.

## Control Flow Notes
This is a Lua binding layer called only when scripts execute ephemera methods. Registration (`Lua_Ephemera_register`) runs once during engine init, installing metamethods and containers. Runtime: Lua script calls e.g. `e:facing(90)`, Lua dispatches to corresponding C function, which accesses/modifies ephemera via `ephemera.h` API. The actual game-world updates (animation, rendering) are driven by core `update_ephemera()` and render system elsewhere.

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h`
- **Game world:** `ephemera.h` (core ephemera functions), `map.h`, `world.h` (coordinate/polygon types)
- **Lua bindings:** `lua_templates.h` (L_Class, L_Enum, L_Container, L_TableFunction), `lua_map.h` (Lua_Polygon, Lua_Collection)
- **Preferences:** `preferences.h` (graphics_preferences ΓåÆ ephemera_quality)
- **Rendering:** `render.h` (TEST_RENDER_FLAG, _polygon_is_visible)
- **Engine state:** Implicit extern functions `remove_ephemera`, `get_ephemera_data`, `new_ephemera`, `set_ephemera_shape`, `remove_ephemera_from_polygon`, `add_ephemera_to_polygon`, `get_dynamic_limit`; macros `DESCRIPTOR_SHAPE_BITS`, `DESCRIPTOR_COLLECTION_BITS`, `MAXIMUM_SHAPES_PER_COLLECTION`, `MAXIMUM_CLUTS_PER_COLLECTION`, `WORLD_ONE`, `FULL_CIRCLE`, `GET_DESCRIPTOR_*`, `GET_OBJECT_SCALE_FLAGS`, `SLOT_IS_USED`
