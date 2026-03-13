# Source_Files/GameWorld/platforms.h - Enhanced Analysis

## Architectural Role

This file provides the **core platform system** for interactive moving geometry in Aleph OneΓÇöa critical subsystem within GameWorld that manages doors, elevators, and other dynamic obstacles. Platforms bridge **geometric state** (polygon/line heights via map.h), **entity pathfinding** (monster accessibility queries), **player interaction** (action key toggles), and **cascading activation** (tag-based chains). The dual-struct design (static vs. runtime data) reflects 1990s-era memory constraints while enabling efficient serialization for savegames and network synchronization.

## Key Cross-References

### Incoming (who depends on this file)

- **GameWorld main loop** (`marathon2.cpp::update_world`): Calls `update_platforms()` once per tick to drive all active platform movement and state transitions
- **Physics/collision system** (`physics.cpp`, `map.cpp`): Calls `monster_can_enter_platform()` and `monster_can_leave_platform()` for pathfinding decisions; uses `PLATFORM_IS_ACTIVE`, `PLATFORM_IS_FULLY_*` macros to query geometry
- **Player interaction** (`player.cpp`, `devices.cpp`): Calls `player_touch_platform_state()` when player presses action key on controllable platform
- **Entity lifecycle** (`placement.cpp`, `items.cpp`): Calls `platform_was_entered()` to trigger activation when player/monster enters polygon
- **Activation triggers** (`devices.cpp`, level startup): Calls `try_and_change_platform_state()` and `try_and_change_tagged_platform_states()` to activate/deactivate by control panel or level logic
- **Media system** (`media.cpp`): Calls `adjust_platform_for_media()` when liquid enters/leaves to recalculate platform accessibility
- **Serialization** (`game_wad.cpp`): Calls `pack_platform_data()` before save, `unpack_platform_data()` after load or network receive
- **Overhead map rendering** (`overhead_map.cpp`): Likely reads platform state flags to draw moving geometry on tactical display

### Outgoing (what this file depends on)

- **map.h**: Imports `polygon_data`, `line_data`, `endpoint_data`, `world_distance`, `MAXIMUM_VERTICES_PER_POLYGON`, `side_data`
- **csmacros.h** (via map.h): `TEST_FLAG16`, `TEST_FLAG32`, `SET_FLAG16`, `SET_FLAG32` for bitfield access
- **Constants** (`WORLD_ONE`, `TICKS_PER_SECOND`): Defines platform speed/delay enums scaled by these; likely from `world.h`
- **XML/MML system** (`InfoTree`): `parse_mml_platforms()` parses platform definitions from scenario XML
- **Global `PlatformList`** (extern vector): Central authority on all platforms in map; accessed via `platforms` macro for C-style compatibility

## Design Patterns & Rationale

**Bitfield macro abstraction (50+ macros):**
Rather than direct bit manipulation (`(p)->static_flags |= (1<<3)`), the code provides `PLATFORM_IS_LOCKED(p)` and `SET_PLATFORM_IS_LOCKED(p, v)` wrappers. This pre-C++11 pattern improves readability and centralizes flag definitions. Trade-off: macro proliferation vs. type safety (no compiler validation of field names).

**Dual data structure strategy:**
- `static_platform_data` (32 bytes): Immutable design template stored in WAD; contains type, speed, delay, min/max heights, flags, polygon, tag
- `platform_data` (140 bytes): Runtime instance extending static data with dynamic state (current heights, flags, endpoints, parent platform)
This mirrors **level-editor semantics** (place a door, define its properties) vs. **runtime state** (door is currently halfway open). Enables compact disk storage while runtime tracks complex state like `endpoint_owner_data` per platform.

**Endpoint owner tracking:**
Each platform embeds `endpoint_owner_data[MAXIMUM_VERTICES_PER_POLYGON]` tracking which polygon endpoints and lines it owns. This is likely for **efficient cascade updates**: when a platform moves, the system updates only relevant geometry, not the entire polygon. Rationale: platforms frequently change polygon heights; endpoint ownership avoids redundant recalculation.

**Tag-based cross-platform activation:**
`try_and_change_tagged_platform_states()` activates all platforms with matching tag. Design rationale: **level designers want grouping** without explicit list definition. Tags allow "all doors on red team" semantics. Pattern mirrors Marathon 1 mission design.

**Serialization abstraction via unpack/pack:**
Rather than embedding serialization logic in struct definitions, separate `unpack_platform_data()` and `pack_platform_data()` functions handle byte conversion with version parameterization. Enables future format evolution without modifying core structs.

## Data Flow Through This File

**Load phase (level start):**
1. WAD file ΓåÆ `unpack_static_platform_data()` ΓåÆ array of `static_platform_data`
2. Each template ΓåÆ `new_platform(static_template, polygon_index)` ΓåÆ allocates `platform_data` entry in global `PlatformList`
3. `adjust_platform_for_media(..., true)` initializes media-aware state (lava/water levels)

**Per-tick update:**
1. `update_platforms()` iterates all platforms in `PlatformList`
2. For each active platform:
   - If extending: increment `floor_height`/`ceiling_height` toward `maximum_floor_height`; check collision with entities
   - If contracting: decrement toward `minimum_floor_height`; check blocking
   - When fully extended/contracted: set positioning flags, trigger adjacent platform chains
3. `adjust_platform_sides()` updates polygon line heights for rendering

**Interaction flow:**
- Player presses action key ΓåÆ `player_touch_platform_state()` ΓåÆ `try_and_change_platform_state()`
- Control panel activation ΓåÆ `try_and_change_platform_states()` iterates all platforms with matching tag
- State change may trigger `_platform_activates_adjacent_platforms_when_activating` ΓåÆ cascades to adjacent platforms

**Media interaction:**
- Liquid height change ΓåÆ `adjust_platform_for_media()` recalculates platform heights, sets `_platform_floor_below_media` flag
- Pathfinding queries check this flag to deny monster access if platform is submerged

**Serialization:**
- Save game: `pack_platform_data(stream, platforms, count)` ΓåÆ byte stream
- Load game: `unpack_platform_data(stream, platforms, count)` ΓåÉ byte stream
- Network sync: same pack/unpack for client-server state reconciliation

## Learning Notes

**1. What this file teaches about the engine:**
- **Early 2000s C codebase**: Extensive use of enums + macros predates C++ templates and flag types. Modern engines use bitfield structs or constexpr flag classes.
- **Immutable design data**: Static templates separate level-editor intent from runtime mutationΓÇöa pattern Marathon 1 made standard for all moving geometry.
- **Tag-based activation chains**: Grouping platforms by tag rather than explicit lists reflects level-designer workflows (edit many objects at once).
- **Dual-buffer world state**: Platforms are updated during `update_platforms()` tick; rendering reads from the same PlatformListΓÇöno separate interpolation buffer (unlike modern engines using predictive rollback).

**2. Idiomatic practices:**
- Heavy reliance on global `PlatformList` vector (not encapsulated in a class)
- Macro-heavy flag access instead of bitfield structs or enums
- Function pointer callbacks not used; instead, flags trigger behaviors directly
- Type constants as enums (0ΓÇô8) rather than string keys or class hierarchy

**3. Modern contrasts:**
- Modern engines (Unity, Unreal) use Components + Events instead of monolithic `platform_data` struct
- Modern C++ would use `std::bitset<32>` or `enum class` with constexpr bitwise ops instead of 50 macros
- Spatial queries (monster pathfinding) would use spatial hashing rather than polygon flood-fill

## Potential Issues

**1. Space inefficiency of endpoint_owner_data:**
Each `platform_data` embeds a fixed `endpoint_owner_data[MAXIMUM_VERTICES_PER_POLYGON]` array. If most platforms move within few polygon endpoints, this wastes ~40+ bytes per platform. Mitigation: Early 2000s memory was tight; space for simplicity was standard tradeoff. Modern engines would use dynamic allocation or separate index table.

**2. Activation loop risk:**
Adjacent platform chain activations (`_platform_activates_adjacent_platforms_when_*` flags) could theoretically cause infinite loops if platform A activates B and B activates A. Code likely prevents this via `_platform_activates_only_once` or other state checks, but the header doesn't enforce it.

**3. Flag saturation:**
With 28 static flags used out of 32 maximum and 10 dynamic flags out of 16, adding new platform types or behaviors requires either bit-shifting existing flags or breaking compatibility. Modern designs would use extensible data (e.g., property maps).

**4. Global PlatformList as authority:**
The extern vector is accessed via `platforms` macro for C-compatibility. This hides allocation failures; if the vector resizes or fails, no caller catches the error. Modern pattern would use optional/result types or assertions.
