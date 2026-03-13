# Source_Files/Files/game_wad.h - Enhanced Analysis

## Architectural Role

`game_wad.h` is the **central save/load orchestration API** bridging the Files subsystem (WAD archives, checksums, file I/O) and GameWorld subsystem (map geometry, entity state, player data). It handles three distinct persistence flows: session-level saves (user save-to-disk), netgame resumption (multi-client state sync), and level transitions (checksum validation). This file is also a policy layerΓÇöit defines *what* gets persisted and *when*, while delegating serialization mechanics to lower-level WAD builders (`build_meta_game_wad`, `process_map_wad`).

## Key Cross-References

### Incoming (who depends on this file)
- **Shell/Interface** (`interface.cpp`, `shell.cpp`): Game initialization, new/load/save/quit lifecycle
- **Network subsystem** (`network_messages.cpp`, `network_games.cpp`): Netgame resume, map sync, player restoration
- **Main loop** (`marathon2.cpp`): Level transitions, state checkpointing, trigger responses
- **Lua/XML** (`XML_MakeRoot.cpp`, Lua bindings): Level script integration queries
- **Dialogs/UI** (preference/save dialogs): User-facing save/load operations

### Outgoing (what this file depends on)
- **Files/WAD subsystem** (`wad.h/cpp`, `FileHandler.h`): Low-level WAD parsing/construction, file I/O, CRC validation
- **GameWorld** (`map.h`, `player.h`, `monsters.h`, `items.h`, `effects.h`, `platform.h`): `wad_data`, `dynamic_data` structures; entity state serialization
- **Utilities** (`crc.h`): Checksum calculation for map integrity
- **Configuration** (implicit): Embedded physics/Lua detection via `level_has_embedded_physics_lua()`

## Design Patterns & Rationale

**1. Serialization Abstraction via Builders**  
The file declares `build_meta_game_wad()` but not the WAD encoding logic itself. This decouples persistence *policy* (what to save) from *mechanism* (how to encode). Allows WAD format evolution without changing this interface.

**2. Dual-Path Data Access**  
`get_dynamic_data_from_save()` (disk I/O, slow) vs. `get_dynamic_data_from_wad()` (memory-resident, fast) provide two paths to identical data. Netgame resume uses the fast WAD path; user load uses the slow disk path. Tradeoff: code duplication vs. flexibility.

**3. Metadata Self-Documentation**  
`save_game_file()` bundles metadata and image alongside game state. Saves are self-contained (including screenshot) without external indexing. Rationale: user-friendly save lists; portable save transfers.

**4. Checksum Validation for Determinism**  
`match_checksum_with_map()` ensures clients load identical map versions. Critical for netgames using lockstep synchronizationΓÇöany state divergence is map mismatch, not code bug.

**5. Version-Gated Deserialization**  
`process_map_wad(..., short version)` accepts a version parameter. Allows backward-compatible loading (v1 save ΓåÆ v2 engine), though specific compatibility logic lives elsewhere.

**6. Gradual API Exposure**  
Comments like "ZZZ: exposed this for netgame-resuming code" and "split this out from new_game" reveal iterative design. Originally monolithic save/load, now decomposed as new use cases emerged (netplay, replay, script integration). Result: somewhat scattered responsibilities.

## Data Flow Through This File

```
SAVE FLOW:
  User triggers save
    ΓåÆ save_game_file(File, metadata, imagedata)
       ΓåÆ build_meta_game_wad(metadata, imagedata, ...)
       ΓåÆ FileSpecifier writes WAD to disk
       
LOAD FLOW (Single-Player):
  User selects save file
    ΓåÆ get_dynamic_data_from_save(File)
       ΓåÆ FileSpecifier reads WAD
       ΓåÆ extract dynamic_data (ticks, RNG seed, counts)
    ΓåÆ process_map_wad(wad, restoring_game=false, version)
       ΓåÆ deserialize map geometry, entities, platforms, lights
       ΓåÆ get_dynamic_data_from_wad(wad, &dest)
       ΓåÆ get_player_data_from_wad(wad)
    ΓåÆ set_map_file(File, runScript=true)
       ΓåÆ optionally execute level Lua/physics scripts
       
NETGAME FLOW:
  Host builds game state snapshot
    ΓåÆ build_save_game_wad(...)
  Clients receive WAD chunk
    ΓåÆ process_map_wad(wad, restoring_game=true, version=netgame_version)
       ΓåÆ match_checksum_with_map(vRefNum, dirID, checksum, File)
       ΓåÆ verify clients have identical map
    ΓåÆ get_player_data_from_wad(wad) for each client
    ΓåÆ sync deterministic RNG seed via dynamic_data
```

**Key constraint:** Deterministic RNG seed must be persisted and restored identically across all clients, or netgame diverges.

## Learning Notes

**1. Determinism-First Architecture**  
The presence of `dynamic_data` (random seed, tick count) and version-gated loading reveals this engine was designed for **deterministic replay and netplay lockstep synchronization**. Modern engines often use client-side prediction with server authority; this engine prioritizes simpler, bit-identical execution across all clients.

**2. Layered Persistence**  
Three levels of save granularity:
- **Full save** (with metadata/screenshot): User save-to-disk
- **WAD snapshot** (game state only): Netgame sync, level transitions, replay
- **Incremental state** (player, dynamic_data separately): Fine-grained restoration

3. **Legacy Encoding Heritage**  
Mixing 1994-era constants (`SAVE_GAME_METADATA_INDEX = 1000`) with modern C++ (`std::string`, `FileSpecifier&`) indicates iterative modernization. The `_new` ΓåÆ `new` C++ naming convention (mentioned in comments) suggests incremental C++11+ adoption.

**4. Map Integrity Validation**  
`match_checksum_with_map()` filesystem search suggests no central map registry. Maps are discovered at runtime; checksums prevent silent loading of wrong versions. Rationale: simplicity (no manifest files), but can be slow on large disk trees.

**5. Embedded Level Customization**  
`level_has_embedded_physics_lua()` reveals levels can override hardcoded physics/AI with per-level Lua scripts or physics models. This is more flexible than static definitions but requires WAD parsing at runtimeΓÇöno pre-computation possible.

## Potential Issues

**1. Version Compatibility Unclear**  
The `version` parameter in `process_map_wad()` suggests multi-version support, but this header doesn't document the compatibility matrix. What if v1.0 save loads in v2.0 engine? Are old dynamic_data fields silently ignored or error-checked?

**2. Global Map Reference Mutations**  
`set_map_file()` and `get_map_file()` manage a global map reference. No thread safety visible. If called out-of-order or during world updates, race conditions or stale references possible. Testing this in isolation is difficult.

**3. Hash Collision Risk**  
`match_checksum_with_map()` performs filesystem search for matching checksum. If two map versions have identical CRC (collision or intentional duplicate), the wrong map loads silently. CRC-32 provides ~1/2^32 collision probability; adequate for small map libraries, risky for large ones.

**4. Script Execution Side Effects**  
`set_map_file(File, runScript=true)` optionally executes Lua scripts. If a script crashes or modifies global state unexpectedly, recovery is unclear. No visible error-handling or rollback mechanism.

**5. Metadata Encoding Unspecified**  
`save_game_file()` takes `std::string metadata` and `imagedata` without specifying encoding (UTF-8? Mac Roman? binary?). Cross-platform save portability depends on undocumented assumptions.
