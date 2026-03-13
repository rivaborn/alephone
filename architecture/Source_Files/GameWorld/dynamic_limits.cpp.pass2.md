# Source_Files/GameWorld/dynamic_limits.cpp - Enhanced Analysis

## Architectural Role

This file acts as a **resource broker and initialization coordinator** for entity pool sizes across the game world. It sits between high-level configuration (film profile version flags and MML XML overrides) and low-level subsystems (effects, monsters, projectiles, objects managers), centralizing all dynamic sizing decisions in one place to prevent scattered allocation logic. The two-phase initialization pattern (preset selection ΓåÆ MML override ΓåÆ reallocation) enables backward compatibility with historical game versions while allowing flexible MML-based customization.

## Key Cross-References

### Incoming (who depends on this file)
- **Entity creation routines**: `monsters.cpp`, `projectiles.cpp`, `effects.cpp` call `get_dynamic_limit()` to query max entity counts before spawning
- **Map/game initialization**: `marathon2.cpp` (or map loader) calls `reset_mml_dynamic_limits()` at game/level startup
- **MML parsing coordinator**: `XML_MakeRoot.cpp` (or equivalent) calls `parse_mml_dynamic_limits()` during XML configuration phase
- **Macro indirection**: Map geometry code uses `MAXIMUM_*_PER_MAP` macros (from `map.h`) which internally call `get_dynamic_limit()` to determine array bounds

### Outgoing (what this file depends on)
- **Global entity pools**: Writes to `EffectList`, `ObjectList`, `MonsterList`, `ProjectileList` (std::vector instances defined in `effects.cpp`, `map.cpp`, `monsters.cpp`, `projectiles.cpp`) via `.resize()`
- **Subsystem allocation**: Calls `allocate_pathfinding_memory()` (from `pathfinding.cpp`) and `allocate_ephemera_storage()` (from `ephemera.cpp`) to trigger dependent allocations
- **Film profile state**: Reads global `film_profile` object (from preferences/game state) to determine compatibility mode
- **Configuration parsing**: Uses `InfoTree` class (from `XML/InfoTree.h`) for MML XML element traversal and attribute extraction

## Design Patterns & Rationale

**Versioned Presets Pattern**: Three static uint16 arrays encode historical limits across game releases (Marathon 2 original, Aleph One 1.0 expanded, Aleph One 1.1 reverted). This reflects an engine that evolved across multiple shipping titles with save-game and demo compatibility requirements. Rather than versioning the entire configuration format, limits are versioned as a lightweight compatibility mechanism tied to `film_profile` flags. This pattern avoids breaking legacy save files while allowing new feature tiers.

**Two-Phase Initialization with Explicit Reallocation Hook**: 
- Phase 1: `reset_dynamic_limits()` selects a preset based on film profile and populates the mutable `dynamic_limits` vector
- Phase 2: `parse_mml_dynamic_limits()` optionally overrides individual limits via XML
- Both phases trigger `reallocate_dynamic_limits()` to sync subsystem memory allocations

This decoupling enables: (a) defaults to work without MML, (b) MML overrides to be composable and optional, and (c) safe reset-to-defaults via `reset_mml_dynamic_limits()`. Explicit reallocation after *every* configuration change prevents the common bug where configured limits diverge from allocated array sizes.

**Defensive Lazy-Initialization Sentinel**: The `get_dynamic_limit()` accessor checks `dynamic_limits_loaded` and falls back to `a1_1_1_dynamic_limits` (a safe default) if queried before initialization. This guards against uninitialized vector reads and allows entity creation code to tolerate pre-initialization queries (though in practice, all queries should happen post-initialization).

**Bounds-Checked Parser Helper**: `parse_limit_value()` encapsulates the repetitive pattern of "find XML child element, extract and clamp uint16 attribute, update array slot." The helper centralizes bounds enforcement [0, 32767] and reduces boilerplate in `parse_mml_dynamic_limits()`, making the parsing logic more maintainable.

## Data Flow Through This File

**Initialization Flow (Startup)**:
```
Game startup
  ΓåÆ reset_mml_dynamic_limits()
      ΓåÆ reset_dynamic_limits()
          [reads film_profile.increased_dynamic_limits_1_1/1_0]
          [selects m2/a1_1_0/a1_1_1 preset]
          ΓåÆ dynamic_limits.assign(preset_array)
          ΓåÆ sets dynamic_limits_loaded = true
      ΓåÆ reallocate_dynamic_limits()
          ΓåÆ EffectList.resize(MAXIMUM_EFFECTS_PER_MAP)
          ΓåÆ ObjectList.resize(MAXIMUM_OBJECTS_PER_MAP)
          ΓåÆ MonsterList.resize(MAXIMUM_MONSTERS_PER_MAP)
          ΓåÆ ProjectileList.resize(MAXIMUM_PROJECTILES_PER_MAP)
          ΓåÆ allocate_pathfinding_memory()
          ΓåÆ allocate_ephemera_storage(dynamic_limits[_dynamic_limit_ephemera])

  ΓåÆ [MML parsing phase]
      ΓåÆ parse_mml_dynamic_limits(InfoTree root)
          [for each limit type: parse_limit_value(..., _dynamic_limit_*)]
              ΓåÆ root.children_named("objects"/"monsters"/...)
              ΓåÆ limit.read_attr_bounded<uint16>("value", dynamic_limits[type], 0, 32767)
          ΓåÆ reallocate_dynamic_limits()  [resizes arrays again with new limits]
```

**Runtime Query Flow (Entity Creation)**:
```
Entity creation in monsters.cpp / projectiles.cpp / effects.cpp
  ΓåÆ max_count = get_dynamic_limit(_dynamic_limit_monsters)
      ΓåÆ if (dynamic_limits_loaded) return dynamic_limits[which]
        else return a1_1_1_dynamic_limits[which]  [safe fallback]
```

**State Transitions**:
- **Pre-init**: `dynamic_limits_loaded = false`, vector uninitialized
- **After reset**: `dynamic_limits_loaded = true`, vector contains preset limits, subsystem arrays sized
- **After MML parse**: Vector may have overridden values, subsystem arrays re-sized to match

## Learning Notes

**Versioning via Global Flags Rather Than File Versioning**: This engine evolved across three distinct releases (Marathon 2, Aleph One 1.0, 1.1) and uses global compatibility flags (`film_profile.increased_dynamic_limits_*`) to drive behavior rather than embedding version info in WAD headers or save files. This is pragmatic but implicitΓÇöa modern engine would use explicit version headers or data versioning. It reflects how 30-year-old codebases grow organically before architectural consolidation.

**Enum-Indexed Flat Array Over Structs**: Limits are stored in a parallel `uint16[]` array indexed by `enum` constants (`_dynamic_limit_monsters`, etc.) rather than a struct with named fields. This C-style pattern (predating widespread OOP in game engines) trades type safety for memory efficiency and cache locality. The cost is fragility: there's no compile-time verification that enum values match array positions.

**Post-Allocation Callbacks Instead of Return Values**: Rather than returning allocated structures, `reallocate_dynamic_limits()` directly invokes subsystem allocation routines (`allocate_ephemera_storage()`, etc.). This avoids circular dependencies but creates implicit couplingΓÇöevery time a new entity subsystem is added, this function must be updated. A more decoupled design would use a callback registry or observer pattern.

**Single Configuration Phase, Not Hot-Reload**: The double call to `reallocate_dynamic_limits()` (once in reset, once in parse) suggests the original author anticipated array reallocation could be expensive or unsafe mid-frame. The design assumes limits are set once at startup, not modified during gameplay. This is typical of deterministic 30 FPS engines where mid-frame state changes can desynchronize replay recordings.

## Potential Issues

**Initialization Order Coupling**: If `get_dynamic_limit()` is called between `reset_mml_dynamic_limits()` and `parse_mml_dynamic_limits()`, it returns base-case limits instead of MML-overridden limits. While likely safe in practice (both are called early in startup), this dependency is undocumented and could break if startup order changes.

**Silent Bounds Clamping Without Warning**: `parse_limit_value()` silently clamps out-of-range values to [0, 32767] via `read_attr_bounded()`. If MML contains `<value>100000</value>`, the limit is clamped without user feedback, risking silent configuration failures. A warning log would help catch configuration errors.

**No Allocation Failure Handling**: `reallocate_dynamic_limits()` calls `.resize()` on global vectors without checking for allocation failure. On memory-constrained systems, a failed resize could leave the world in an inconsistent state (limits say 512 monsters, but `MonsterList` only has 256 allocated slots), causing crashes during entity creation.

**Fragile Enum-to-Index Mapping**: The file does not directly include the header defining `_dynamic_limit_*` enum values, so there's no compile-time verification that enum order matches array layout. Reordering enum values in the header will silently corrupt which limits are read, causing difficult-to-diagnose bugs.

**Implicit Thread Unsafety**: The file-static `dynamic_limits_loaded` boolean has no synchronization primitives. In a multithreaded context (e.g., if background threads query limits while the main thread resets), a race condition is possible. This is likely not an issue in the deterministic single-threaded 30 FPS game loop, but it's not formalized.
