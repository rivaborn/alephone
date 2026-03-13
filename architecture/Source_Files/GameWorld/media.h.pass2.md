# Source_Files/GameWorld/media.h - Enhanced Analysis

## Architectural Role

Media is a per-polygon environmental subsystem enabling configurable liquid simulation (water, lava, goo, sewage, Jjaro) with physics effects, damage mechanics, and audio cues. It bridges the map geometry layer (polygons, lights) with entity/physics systemsΓÇöproviding damage calculations for monsters/players, sound triggers for footsteps/splashes, and visual effects. Media integrates tightly into the frame loop via `update_medias()` and participates in save/network synchronization via binary serialization.

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld/marathon2.cpp**: Calls `update_medias()` once per frame during `update_world()` loop orchestration
- **GameWorld/monsters.cpp**: Queries `IsMediaDangerous(media_type)` to decide if monster should navigate into media
- **GameWorld/player.cpp / projectiles.cpp**: Polls `get_media_damage()`, `get_media_sound()` on collision/entry
- **GameWorld/platforms.cpp**: Calls `adjust_platform_for_media()` to cascade height changes when media height changes
- **Sound/SoundManager.cpp**: Fetches media sound indices via `get_media_sound(media_index, event_type)` for 3D positioning
- **Files/game_wad.cpp**: Serializes media via `pack_media_data()` / `unpack_media_data()` for save/network
- **RenderMain/render.cpp**: Reads `texture` and `transfer_mode` fields for media appearance
- **Lua bindings**: Likely expose media accessors for scripted environmental effects

### Outgoing (what this file depends on)
- **map.h**: Provides `SLOT_IS_USED`, polygon/light definitions, `damage_definition` struct, world coordinate types
- **world.h / type system**: `world_distance`, `world_point2d`, `angle`, `_fixed`, `shape_descriptor`, `transfer_mode`
- **XML/InfoTree**: MML parser for `parse_mml_liquids(root)` customization

## Design Patterns & Rationale

### 1. **Light-Height Mapping (Clever Height Deformation)**
Media height is **not** staticΓÇöit's computed dynamically from a light's intensity: `height = low + (high - low) ├ù intensity`. This avoids per-frame recalculation and leverages the existing light animation system to create flexible, animated water surfaces. Trade-off: **requires monotonic light curves** (strobes will break it). This is a ~20-year optimization choice documented in the 2000 comments; modern engines compute height via geometry or shaders.

### 2. **Vector-Based Dynamic Allocation**
Evolved from fixed `#define MAXIMUM_MEDIAS_PER_MAP 16` to `vector<media_data>`. Backward-compatibility macros `medias` and `MAXIMUM_MEDIAS_PER_MAP` mask the transition, allowing `.cpp` files to remain unchanged. This is a safe migration pattern for legacy codebases.

### 3. **Safe Accessor Pattern**
`get_media_data(size_t index)` **returns null for out-of-bounds indices** rather than crashing. Encourages defensive null-checking; avoids bounds assertions in hot paths (per Loren Petrich's 2000 note).

### 4. **Query Method Proliferation**
Rather than returning the full `media_data` struct, separate query functions (`get_media_sound()`, `get_media_damage()`, `get_media_detonation_effect()`) allow:
- **Late binding**: Effects can be overridden by MML without changing code
- **Lazy evaluation**: Only compute what's needed (damage rarely queried for non-entity collisions)
- **Encapsulation**: Callers don't need to know struct layout

### 5. **Binary Serialization Support (Network + Save)**
`pack_media_data()` / `unpack_media_data()` enable deterministic sync for multiplayer and save-game restoration. 32-byte struct size is critical for compact I/O.

## Data Flow Through This File

**Inflow:**
1. Map loading reads media definitions from WAD file
2. `new_media()` allocates and registers each media in `MediaList`
3. MML parsing via `parse_mml_liquids()` patches media properties at runtime

**Processing:**
1. Each frame, `update_medias()` updates all media state (animation, effects)
2. When polygon heights change, adjacent media recalculates via light coupling

**Outflow:**
- **Physics**: Entities query `get_media_damage()` on collision; pathfinding checks `IsMediaDangerous()`
- **Audio**: Sound system fetches event-based sounds (`_media_snd_feet_entering`, etc.)
- **Rendering**: Texture + transfer mode drive visual appearance
- **Persistence**: `pack_media_data()` encodes for network sync / save games

## Learning Notes

1. **Light Intensity as a Data Channel**: Using light intensity to drive media height is idiomatic to 1990s Marathon engine design. Modern engines parameterize geometry directly; this shows how constrained systems repurpose existing subsystems (lights ΓåÆ heights).

2. **Incremental Extensibility via MML**: The header declares `parse_mml_liquids(InfoTree&)`, allowing designers to define custom media without C++ code. This is a precursor to modern data-driven design; the XML tree likely overrides texture, damage, and sounds.

3. **Minimal Struct, Maximal Queries**: The 32-byte constraint (likely a performance optimization from the Mac port era) forces separation of concernsΓÇöqueries are thin wrappers around a compact core, not fat getters.

4. **Frame-Loop Integration**: `update_medias()` is called once per 30 FPS tick, tightly coupled to the game loop's orchestration in `marathon2.cpp`. No independent threading or lazy updates.

## Potential Issues

1. **Strobing Light Caveat (Documented but Not Enforced)**
   The comment warns: *"clearly discontinuous light functions (e.g., strobes) should not be used."* No validation prevents this at load time; buggy level designs will silently produce flickering water heights.

2. **32-Byte Structure Rigidity**
   Two unused `int16` slots are already present. Adding new fields (e.g., viscosity, temperature) requires careful padding. Future extensibility is constrained.

3. **No Explicit Bounds Checking**
   While `get_media_data()` null-checks, functions like `get_media_sound(media_index, type)` accept indices without visible validation in the header. Callers must trust correct indices from `new_media()` return values.

4. **Circular Dependency with map.h**
   Both include each other (media.h includes map.h, and map.h likely includes media.h for platform-media interactions). Not critical, but reflects the tight coupling of geometry and fluid systems in this engine.
