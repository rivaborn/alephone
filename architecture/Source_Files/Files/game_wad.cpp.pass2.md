# Source_Files/Files/game_wad.cpp - Enhanced Analysis

## Architectural Role

`game_wad.cpp` is the **critical orchestrator between Files and GameWorld subsystems**, managing bidirectional serialization of complete game state. It's the sole point where game world entities (monsters, projectiles, effects, players, platforms) transition between transient memory and persistent WAD archives. This file is essential for three distinct game flows: (1) new game initialization and level loading, (2) save game persistence and restoration, and (3) networked multiplayer map distribution and state synchronization.

## Key Cross-References

### Incoming (who depends on this file)
- **Interface/Shell** (`interface.cpp`): calls `goto_level()` for level transitions and `new_game()` to start campaigns
- **Network subsystem** (`network.cpp`, `network_messages.cpp`): calls `get_map_for_net_transfer()` for map distribution and `process_net_map_data()` to receive peer maps
- **Save game system** (implicitly via UI layer): triggers `build_save_game_wad()` serialization
- **XML/Scripting** (`XML_LevelScript.h`): integrated into `goto_level()` via `RunLevelScript()`
- **Plugins** (via `Plugins::instance()`): checksum registration in `set_map_file()`
- **Preferences** (via `get_dynamic_data_from_save()`): loads save game metadata without full WAD processing

### Outgoing (what this file depends on)
- **Files subsystem** (`wad.h`, `FileHandler.h`, `Packing.h`): binary I/O, WAD format parsing, chunk serialization
- **GameWorld entities** (`map.h`, `monsters.h`, `projectiles.h`, `effects.h`, `player.h`, `platforms.h`): unpacks into global entity lists; reads state for packing
- **Rendering** (`render.h`, `ChaseCam.h`): allocates render memory, initializes chase cam
- **Audio** (`SoundManager.h`, `Music.h`): loads sound definitions from map
- **Lua/Scripting** (`XML_LevelScript.h`): loads and executes level scripts as part of level entry
- **World state queries** (`flood_map.h`, `lightsource.h`, `scenery.h`): finalizes level via platform/scenery scanning
- **Error handling** (`game_errors.h`): tracks and reports file/format errors

## Design Patterns & Rationale

| Pattern | Evidence | Rationale |
|---------|----------|-----------|
| **Factory** | `build_save_game_wad()`, `build_export_wad()` create WADs from current game state | Isolates WAD construction logic from caller; enables multiple export formats (save vs. map export) |
| **Visitor / Switch Dispatch** | `tag_to_global_array_and_size()` giant switch over 30+ chunk types | Each WAD chunk type requires different pack/unpack handler; maintains type-count correspondence |
| **State Snapshot** | `revert_game_data` struct captures pre-level game state | Enables rollback on level load failure without full world reload |
| **Lazy Capture** | `static_platforms` vector saved/restored during export | Map export must preserve platform state from initial load; snapshots avoid re-computing |
| **Version Adapter** | `version` parameter threaded through all `load_*()` functions | Marathon 1 save format differs from Infinity; handlers conditional on version |
| **Dual-Path Serialization** | `save_data[]` vs. `export_data[]` metadata arrays | Full game save includes dynamic state (players, monsters, effects); map export only initial geometry |

**Why this structure?** The file evolved across ~20 years (1994ΓÇô2002+ comments) as save formats changed and features (Pfhortran scripts, physics models, object-oriented I/O) were added. Heavy use of manual packing/unpacking reflects pre-C++11 serialization practices; modern code would use protobuf or msgpack.

## Data Flow Through This File

**Load Path (Level ΓåÆ Memory):**
```
MapFileSpec ΓåÆ open_wad_file_for_reading()
    Γåô
read_wad_header() ΓåÆ validate version
    Γåô
read_indexed_wad_from_file() ΓåÆ raw chunks (polygon, side, line, object, platform, light, etc.)
    Γåô
process_map_wad() ΓåÆ dispatcher
    Γö£ΓåÆ allocate_map_for_counts() ΓåÆ pre-size all geometry vectors
    Γö£ΓåÆ load_points/lines/sides/polygons() ΓåÆ unpack into global EndpointList, LineList, etc.
    Γö£ΓåÆ load_objects() ΓåÆ place monsters, items, projectiles
    Γö£ΓåÆ load_platforms() ΓåÆ deserialize platform state
    Γö£ΓåÆ RunLevelScript() ΓåÆ execute MML/Lua customizations
    ΓööΓåÆ complete_loading_level() ΓåÆ finalize: redundant map data, scenery, lightsource indices
        Γåô
    entering_map() ΓåÆ player/world state finalization (UI layer)
```

**Save Path (Memory ΓåÆ File):**
```
Current game state (globals: ObjectList, MonsterList, EffectList, ProjectileList, PlatformList, etc.)
    Γåô
build_save_game_wad()
    Γö£ΓåÆ Iterate save_data[] array (30+ chunk types)
    ΓööΓåÆ For each type: tag_to_global_array_and_size()
        Γö£ΓåÆ pack_*_data() ΓåÆ serialize entities into byte buffer
        ΓööΓåÆ append_data_to_wad() ΓåÆ add chunk to WAD
    Γåô
write WAD to file ΓåÆ save game persisted
```

**Network Path (Multiplayer):**
```
get_map_for_net_transfer(entry_point) ΓåÆ get_flat_data(MapFileSpec) ΓåÆ compress/serialize
    Γåô (network transport)
process_net_map_data() ΓåÆ inflate_flat_data() ΓåÆ process_map_wad(restoring_game=false)
    Γåô (all players synchronized to shared map state)
```

**Restore Path (Load Saved Game):**
```
load_level_from_map(level_index=NONE) ΓåÆ flags restoring_game=true
    Γåô
process_map_wad(..., restoring_game=true) ΓåÆ unpack initial map + saved game chunks
    Γö£ΓåÆ get_player_data_from_wad() ΓåÆ restore player inventory, state, position
    Γö£ΓåÆ restore all monsters/projectiles/effects from save chunks
    ΓööΓåÆ complete_restoring_level() ΓåÆ finalize (calls complete_loading_level logic)
```

## Learning Notes

**Engine Architecture Legacy:**
- This file spans ~2800 lines of manual binary I/O predating JSON/Protobuf. The `revert_game_info` struct and version-conditional loading reflect iterative protocol evolution over 25+ years.
- Comments (Loren Petrich 2000, Br'fin 2002) show collaborative maintenance; each contributor tagged their changes. Modern codebases would use VCS blame.
- Platform list dual-path handling (`_platform_data` vs. `actual_platform_data`) indicates mid-refactor code that preserves backwards compatibility while supporting new serialization.

**Serialization Patterns:**
- Heavy use of `StreamToValue()` and manual byte-offset tracking in `load_*()` handlers ΓÇö no structured serialization library. Each entity type requires custom pack/unpack pair.
- `pack_directory_data()` / `unpack_directory_data()` abstracts the WAD's directory chunk (level entry points). Demonstrates attempt to centralize format knowledge.

**Network Multiplayer Design:**
- Map is distributed via `get_flat_data()` + compression (likely zlib). All peers must have identical map to prevent desyncs.
- Network saved-game restoration via `process_net_map_data()` ensures mid-game join doesn't require full map re-distribution.

## Potential Issues

| Issue | Location | Risk |
|-------|----------|------|
| **Version branching scattered** | `load_sides()`, `load_polygons()`, `load_objects()`, etc. | If version format changes, multiple functions must update in sync. No central version dispatch. |
| **Dual platform data paths** | `complete_loading_level()`: `if(_platform_data \|\| (_platform_data==NULL && actual_platform_data==NULL))` | Confusing ternary logic; hard to verify correctness. Should clarify intent via named enum. |
| **Manual memory lifecycle** | `tag_to_global_array_and_size()` returns `new[]` buffer; callers must `delete[]`. | String of "delete[] buf" calls post-append could leak if exception thrown during append. |
| **Large static metadata arrays** | `export_data[]`, `save_data[]` ΓÇö ~30+ entries each | If entity types added (new monster, weapon), must append corresponding entries. Easy to miss. |
| **Revert state incomplete** | `revert_game_data` captures `game_data`, `player_start_data`, `entry_point` only | Doesn't capture current inventory, health, ammo, Lua state. Rollback may lose player progress. |
| **Network map sync single-path** | `get_map_for_net_transfer()` always uses `false` for union maps | If union maps (multiple sources) needed for network play, design may not support it. |
