# Source_Files/GameWorld/dynamic_limits.h - Enhanced Analysis

## Architectural Role

This header bridges **GameWorld entity management** with the **XML/MML configuration subsystem**, defining the interface through which engine limits are initialized from modding configurations and queried at runtime. The enum codifies Marathon's resource capping strategy: by capping objects, monsters, projectiles, and collision buffers separately, the engine throttles spawn rates and simulation work without destroying entities mid-frame. This is critical for the 30 FPS deterministic world loop in `marathon2.cpp`, which must respect these budgets during entity allocation in `placement.cpp` and active entity cycling in `monsters.cpp`, `projectiles.cpp`, `effects.cpp`.

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld entity lifecycle** (`placement.cpp`, `monsters.cpp`, `projectiles.cpp`, `items.cpp`, `effects.cpp`, `ephemera.cpp`): Call `get_dynamic_limit()` to enforce spawn caps before creating new entities
- **Pathfinding** (`pathfinding.cpp`, `flood_map.cpp`): Respect `_dynamic_limit_paths` when allocating waypoints
- **Rendering** (`RenderPlaceObjs.h`): Respects `_dynamic_limit_rendered` to cull object counts
- **Map loading** (`game_wad.cpp`): Pre-allocates entity storage based on limit values after parsing

### Outgoing (what this file depends on)
- **XML subsystem** (`InfoTree` from `XML/`): Parsed configuration tree passed to `parse_mml_dynamic_limits()`
- **CSeries types** (`cstypes.h`): `uint16` return type, `int` enum parameter
- **Implicit**: Implementation file (not shown) holds the backing storage and default values

## Design Patterns & Rationale

**Configuration-driven constraints**: Rather than compile-time constants, limits are runtime-configurable via MML, enabling modders to tune memory/performance for their custom scenarios without recompilation. This reflects Marathon's mod-first design philosophy from the 1990s.

**Enum-based type dispatch**: Each limit has a symbolic `_dynamic_limit_*` constant, avoiding fragile string-key lookups in tight loops. Modern engines often use string keys for flexibility; this enum pattern is faster but less extensible.

**Lazy initialization / reset pattern**: Separation of `parse_mml_dynamic_limits()` (one-time init) and `reset_mml_dynamic_limits()` (scenario transitions) suggests limits are stored in static/global state, re-parsed on level load but cached for query-time performance.

## Data Flow Through This File

1. **Startup**: Engine bootstrap ΓåÆ XML parsing subsystem ΓåÆ `parse_mml_dynamic_limits(root)` ΓåÆ internal limit storage populated (location: `.cpp` file, hidden from header)
2. **Entity spawning**: GameWorld managers call `get_dynamic_limit(_dynamic_limit_objects)` ΓåÆ check current count vs. limit ΓåÆ defer spawn if at cap
3. **Level transition**: New scenario loaded ΓåÆ `reset_mml_dynamic_limits()` ΓåÆ parse new config ΓåÆ entity allocators see updated limits
4. **Collision buffers**: Two special cases (`_dynamic_limit_local_collision` [16], `_dynamic_limit_global_collision` [64]) suggest pre-allocated fixed-size ring buffers for per-frame collision queries, not dynamic pools

## Learning Notes

**Era-specific architecture**:
- This file encapsulates Marathon's constraint model: fixed budgets (not dynamic pools) for collision, effects, and garbage (corpses). Modern engines use dynamic allocation + memory pools.
- The `_dynamic_limit_garbage` / `_dynamic_limit_garbage_per_polygon` split reveals level-design considerations: preventing corpse spam map-wide *and* in tight areas, preventing framerate collapse from overdraw.
- XML-driven limits enabled community modding; compare to hardcoded limits in early 1990s id games.

**Enum layout insight**: The progression from high-level (objects, monsters) to low-level (local/global collision buffers, ephemera) mirrors the entity update order in `marathon2.cpp`'s main loop.

## Potential Issues

- **No bounds validation**: `get_dynamic_limit(int which)` has no range check; passing an invalid enum index (or typo) will read undefined memory or crash. Implementation should bounds-check or assert.
- **uint16 ceiling**: Limits capped at 65535 entities; modern maps with thousands of decorative objects may chafe against this.
- **Collision buffer hardcoding**: `[16]` and `[64]` magic numbers in enum comments suggest these are NOT runtime-configurable despite the `get_dynamic_limit()` interface; the implementation may ignore MML overrides for these critical buffers.
- **Silent cap behavior**: No callback/event when limits hit; spawning managers must actively query and handle refusal; silent failure could mask design errors in mods.
