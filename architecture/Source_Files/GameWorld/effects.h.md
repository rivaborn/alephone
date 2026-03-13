# Source_Files/GameWorld/effects.h

## File Purpose
Defines the interface for managing in-world visual effects (explosions, blood splashes, projectile contrails, teleportation effects, etc.). Provides effect creation, update, and lifecycle management with dynamic limits and serialization support for saving/loading game state.

## Core Responsibilities
- Define 70+ effect types (weapons, blood, environment interactions, creature-specific)
- Manage active effects via dynamic vector storage with runtime-adjustable limits
- Provide effect instance lifecycle: creation, per-frame updates, removal
- Handle effect persistence flags and activation delays
- Serialize/deserialize effect data and definitions for save files and M1 compatibility

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| effect types (enum) | enum | 70+ effect type IDs (explosions, blood, contrails, splashes, sparks, etc.) |
| `effect_data` | struct | 32-byte instance data: type, object reference, flags, special data, delay |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `EffectList` | `std::vector<effect_data>` | global | All currently-active effects in the world |
| `MAXIMUM_EFFECTS_PER_MAP` | macro | global | Dynamic limit from `get_dynamic_limit(_dynamic_limit_effects)` |

## Key Functions / Methods

### new_effect
- Signature: `short new_effect(world_point3d *origin, short polygon_index, short type, angle facing)`
- Purpose: Create and register a new effect instance in the world
- Inputs: 3D origin position, containing polygon, effect type ID, facing angle
- Outputs/Return: Effect index (short) for future reference
- Side effects: Adds to `EffectList`, respects `MAXIMUM_EFFECTS_PER_MAP` limit
- Calls: (defined elsewhere)
- Notes: Fails silently if effect list is full

### update_effects
- Signature: `void update_effects(void)`
- Purpose: Advance all active effects by one game tick
- Inputs: None (operates on global `EffectList`)
- Outputs/Return: None
- Side effects: Updates effect state, animation frames, visibility; removes expired effects
- Calls: (defined elsewhere)
- Notes: Called once per frame; assumes 1 tick elapsed

### remove_effect
- Signature: `void remove_effect(short effect_index)`
- Purpose: Immediately remove an effect from the world
- Inputs: Effect index
- Outputs/Return: None
- Side effects: Marks slot in `EffectList` as free
- Calls: (defined elsewhere)

### get_effect_data
- Signature: `effect_data *get_effect_data(const short effect_index)`
- Purpose: Accessor for effect instance data by index
- Inputs: Effect index
- Outputs/Return: Pointer to `effect_data` struct
- Side effects: None
- Notes: Inline function per comment; bounds checking handled elsewhere

### Serialization functions
- `unpack_effect_data()`, `pack_effect_data()`: Instance data I/O
- `unpack_effect_definition()`, `pack_effect_definition()`: Effect definition I/O  
- `unpack_m1_effect_definition()`: Marathon 1 backward compatibility
- `init_effect_definitions()`: Load definitions from resources

### Utility functions
- `teleport_object_in()`, `teleport_object_out()`: Special teleportation effects
- `remove_all_nonpersistent_effects()`: Clear transient effects (e.g., on level restart)
- `mark_effect_collections()`: Mark effect resources for loading/unloading

## Control Flow Notes
Effects follow a standard game loop pattern: created on-demand (via `new_effect()`), updated each frame (via `update_effects()`), and removed when expired or explicitly cleared. The delay field allows effects to be invisible/inactive for N ticks before becoming visible.

## External Dependencies
- **Includes:**
  - `world.h` ΓÇô world coordinate types (`world_point3d`, `angle`) and spatial primitives
  - `dynamic_limits.h` ΓÇô `get_dynamic_limit()` for runtime effect cap
  - `<vector>` ΓÇô C++ standard library dynamic array
- **Defined elsewhere:** Effect definition structures, implementations of all declared functions, resource loading
