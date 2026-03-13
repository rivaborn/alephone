# Source_Files/GameWorld/scenery.cpp - Enhanced Analysis

## Architectural Role

Scenery.cpp bridges the **Object System** (map.h) with **Game World Configuration** (MML/XML) to manage non-player, non-monster decorative objects. It acts as a thin specialization layer: scenery objects are created as standard map objects with `_object_is_scenery` owner flag, then managed through animation lists and destruction state machines. This lightweight abstraction lets the core object system remain generic while providing scenery-specific lifecycle hooks (damage transition, animation population, MML reconfiguration).

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld/marathon2.cpp**: Calls `animate_scenery()` each game loop tick (main update orchestration)
- **GameWorld/map.cpp** or **physics.cpp**: Calls `damage_scenery()` when scenery takes damage/explosions
- **Files/game_wad.cpp**: Calls `new_scenery()` during map loading to instantiate scenery objects from WAD data
- **GameWorld/placement.cpp**: Likely calls `new_scenery()` during level placement (inferred from WAD loading)
- **RenderMain/render.cpp** or callers: May call `get_scenery_collection()` / `get_damaged_scenery_collection()` to fetch shape/texture descriptors for rendering
- **GameWorld/map.cpp** or **lightsource.cpp**: Calls `get_scenery_dimensions()` for collision/spatial queries

### Outgoing (what this file depends on)
- **map.h**: `new_map_object()`, `get_object_data()`, `animate_object()`, `randomize_object_sequence()`, `MAXIMUM_OBJECTS_PER_MAP`, `SLOT_IS_USED()`, object flag macros
- **effects.h**: `new_effect()` to spawn destruction visual/audio effects
- **scenery_definitions.h**: External array `scenery_definitions[]` and constant `NUMBER_OF_SCENERY_DEFINITIONS` (defines all scenery properties)
- **InfoTree.h**: MML/XML parsing for `parse_mml_scenery()` configuration override

## Design Patterns & Rationale

### 1. **Lightweight Object Specialization**
Scenery objects are *not* a distinct typeΓÇöthey're regular map objects with `_object_is_scenery` ownership flag. This reuses all object infrastructure (shape, location, animation sequence, permutation) without duplication. Trade-off: scenery properties live in a parallel `scenery_definitions[]` array keyed by `object->permutation`, rather than embedded in the object struct. This trades a lookup per operation for memory efficiency (objects are already large).

### 2. **Opt-In Animation via Tracking List**
Not all scenery animatesΓÇö`randomize_object_sequence()` returns *false* only for animated objects, which are then added to `AnimatedSceneryObjects` vector. This avoids per-object animation state and per-object iteration of static scenery. The small static capacity (`MAXIMUM_ANIMATED_SCENERY_OBJECTS=20`, typically << total scenery count) reflects level design practices.

### 3. **Deferred Solidity Correction (MML Timing Workaround)**
The engine creates scenery objects at WAD load time, but MML configuration (user overrides) loads *after*. The `ok_to_reset_scenery_solidity` flag + `reset_scenery_solidity()` function solve this: MML parse sets the flag, then reapplies solidity to all existing scenery. This is a **band-aid for initialization order fragility**ΓÇöa cleaner design would defer object creation until after config, but that's architecturally coupled to level loading.

### 4. **Configuration Backup/Restore Pattern**
`original_scenery_definitions` snapshot allows `reset_mml_scenery()` to undo MML changes without keeping multiple file copies. Allocate-on-first-use pattern ensures overhead only if MML is actually parsed. Trade-off: malloc-managed backup can leak if reset is never called.

### 5. **Private Definition Lookup**
`get_scenery_definition()` wraps unsafe array bounds checking via `GetMemberWithBounds()`, forcing all consumers through a safe interface. Returns `NULL` for out-of-range types; callers must check. This prevents index buffer overruns at the cost of null-check boilerplate.

## Data Flow Through This File

```
MAP LOAD PHASE:
  game_wad.cpp::load_level
    ΓåÆ placement.cpp::_recreate_objects
      ΓåÆ new_scenery(location, type)
        ΓåÆ new_map_object() ΓåÆ [creates map object]
        ΓåÆ SET_OBJECT_OWNER(_object_is_scenery)
        ΓåÆ SET_OBJECT_SOLIDITY(flags from definition)
  
  XML::parse_mml_scenery()
    ΓåÆ [backs up original_scenery_definitions]
    ΓåÆ [parses "object" elements, updates scenery_definitions[]]
    ΓåÆ reset_scenery_solidity()
      ΓåÆ [iterates all objects, reapplies solidity from updated defs]

GAME LOOP (per tick):
  marathon2.cpp::update_world
    ΓåÆ animate_scenery()
      ΓåÆ [for each index in AnimatedSceneryObjects]
        ΓåÆ animate_object(index)  [advances animation frame]

DAMAGE/DESTRUCTION:
  physics.cpp or combat system
    ΓåÆ damage_scenery(object_index)
      ΓåÆ [if _scenery_can_be_destroyed flag set]
        ΓåÆ object->shape = destroyed_shape
        ΓåÆ [optionally randomize destroyed animation]
        ΓåÆ new_effect(destroyed_effect) [spawn explosion/sound]
        ΓåÆ SET_OBJECT_OWNER(_object_is_normal)  [no longer scenery]

QUERIES (for rendering/collision):
  render.cpp or spatial queries
    ΓåÆ get_scenery_dimensions(type) ΓåÆ [fetch radius/height]
    ΓåÆ get_scenery_collection(type) ΓåÆ [fetch shape descriptor for rendering]
```

**Key insight**: Scenery is **stateless at the object level**ΓÇöall properties come from `scenery_definitions[]` lookup. The only per-object state is position, facing, animation sequence, and shape (which changes on destruction).

## Learning Notes

### Idiomatic to 1990s/2000s Game Engines (Aleph One era)

1. **Explicit pool management**: `AnimatedSceneryObjects` vector is manually populated/depopulated. Modern engines use component systems or ECS where iteration is automatic. The explicit list reflects era-appropriate structure: tight control over iteration order, cache locality, minimal per-entity overhead.

2. **Permutation as type identifier**: Using `object->permutation` field to store scenery type index is a clever bit-packing trick (permutation field originally meant animation frame variant). Avoids enlarging the object struct at the cost of semantic coupling.

3. **Bounds-checked array access**: `GetMemberWithBounds()` rather than a hash map or vectorΓÇöassumes scenery type is a small int. Efficient but fragile if scenery types become sparse.

4. **Global configuration arrays**: `scenery_definitions[]` is a global array indexed by type. Modern engines use asset registries or resource managers. This reflects Marathon's era-appropriate memory model: fixed-size typed resource collections, not dynamic allocation.

5. **MML as **late-stage modifier**: MML (Marathon Map Language) is applied *after* initial world state setup, requiring workarounds like `ok_to_reset_scenery_solidity`. Modern engines bake config into asset pipelines.

### Modern Contrast

- **Component/ECS systems** would attach animation, physics, and visual components to scenery entities dynamically, avoiding special-case iteration.
- **Asset-driven initialization** would populate scenery from config *before* object creation, not after.
- **Configuration validation** would happen at parse time (MML), not deferred with reset flags.

## Potential Issues

1. **Linear search in `deanimate_scenery()`** (line 125ΓÇô128):
   - O(n) vector iteration to find and erase. For MAXIMUM_ANIMATED_SCENERY_OBJECTS=20, acceptable, but not scalable.
   - **Fix**: Use a `std::unordered_set<short>` or `std::set<short>` for O(log n) removal if scenery counts grow.

2. **No validation of `destroyed_shape` in `damage_scenery()`** (line 180):
   - Blindly assigns `object->shape = definition->destroyed_shape` without checking if the shape descriptor is valid.
   - **Risk**: If MML corrupts or sets an invalid destroyed_shape index, rendering will try to access an out-of-bounds shape, crashing or displaying garbage.
   - **Fix**: Add a bounds check: `if (definition->destroyed_shape is valid) { ... }`

3. **Potential memory leak in MML configuration** (line 235ΓÇô240):
   - `original_scenery_definitions` is allocated on first `parse_mml_scenery()` call and freed in `reset_mml_scenery()`.
   - **Risk**: If `parse_mml_scenery()` is called but `reset_mml_scenery()` is never called (e.g., app crash, scenario switching without cleanup), the backup is leaked.
   - **Fix**: Use `std::unique_ptr<scenery_definition[]>` to auto-free on scope exit, or call reset during level unload.

4. **Global `ok_to_reset_scenery_solidity` flag is fragile** (line 165):
   - External code must set this flag before MML parsing, then calling functions must be aware of the ordering.
   - **Risk**: If level initialization order changes (e.g., MML parsed before scenery created), the flag system breaks silently.
   - **Fix**: Move MML parsing to *before* object creation, eliminating the need for post-hoc solidity reset.

5. **Comment "UGH" indicates known architectural debt** (line 163):
   - The comment itself signals that the developers knew this was a workaround, not a clean solution.
   - Suggests this area is a known brittleness point.

---

## Summary

Scenery.cpp is a **thin specialization layer** that demonstrates good separation of concerns (scenery = objects + definitions + animation tracking) but carries **era-appropriate architectural cruft** (deferred MML loading, global state flags, manual list management). The code is **correct and performant** for its design constraints (small fixed scenery counts, 30 FPS tick rate), but would benefit from **modernization** (ECS, earlier config binding, STL containers for dynamic scenarios).
