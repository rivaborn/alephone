# Source_Files/XML/QuickSave.h - Enhanced Analysis

## Architectural Role

QuickSaves is a **save game lifecycle manager** sitting at the boundary between the Files subsystem (which handles WAD serialization via `game_wad.cpp`) and the UI/game loop that needs to present and manipulate save lists. It acts as a scoped in-memory cache of enumerated saves, with responsibility for pruning and metadata tracking. The singleton pattern reflects that only one save collection should exist per session, but the global helper functions (`create_quick_save`, `delete_quick_save`, etc.) route actual I/O through the Files layer.

## Key Cross-References

### Incoming (who depends on this file)

- **UI/Shell:** Calls `enumerate()` to populate save dialogs, `load_quick_save_dialog()` for file picker, `delete_quick_save()` for manual deletion
- **Game save flow:** Calls `create_quick_save()` after level completion or manual save
- **Save pruning:** Calls `delete_surplus_saves(max_saves)` during periodic cleanup (likely in preferences or on shutdown)
- **Game reloads:** Calls `load_quick_save_dialog()` ΓåÆ `load_quick_save()` to restore state

### Outgoing (what this file depends on)

- **Files subsystem** (`FileHandler.h`): `FileSpecifier` abstracts cross-platform save paths and opening
- **Files subsystem** (`game_wad.cpp`): `create_quick_save()` likely wraps `build_save_game_wad()` + CRC validation; `delete_quick_save()` unlinks filesystem
- **Files subsystem** (`crc.h`): Probably uses `calculate_crc_for_file()` to validate saved-game integrity (not visible in header, but idiomatic for this era)
- **Implicit:** Likely calls `build_map_preview()` (defined in `QuickSave.cpp` per cross-reference index) to generate thumbnail metadata

## Design Patterns & Rationale

| Pattern | Evidence | Rationale |
|---------|----------|-----------|
| **Singleton** | `static QuickSaves* instance()`, private ctor | Only one save collection per session; persists across UI interactions; simplifies global access in game loop |
| **Lazy metadata formatting** | `formatted_time`, `formatted_ticks` stored in struct | UI-centric optimization: format once during enum, not repeatedly during rendering; reduces hot-path work |
| **Comparison operator** | `operator<` on `save_time` | Enables age-based pruning; `std::sort()` in `delete_surplus_saves()` likely uses this to identify oldest saves |
| **Friend class** | `friend class QuickSaveLoader` | `QuickSaveLoader` is privileged helper with direct vector access (probably in .cpp); separates concerns: QuickSaves manages collection, QuickSaveLoader handles deserialization |
| **Container facade** | `begin()`, `end()` exposing STL iterators | Allows range-based iteration and standard algorithms without exposing `m_saves` directly |

**Tradeoff:** The struct mixes _data_ (file path, ticks, player count) with _metadata_ (formatted strings). This violates single responsibility but optimizes for the UI layer's needsΓÇöa common pragmatic choice in game engines.

## Data Flow Through This File

```
ENUMERATION PHASE:
  [Filesystem] --[enumerate() scans dirs]--> [m_saves vector]
                                               (sorted by save_time)
                                               (formatted_time pre-computed)

CREATION PHASE:
  [Game state] --[create_quick_save()]---> [game_wad::build_save_game_wad()]
                                           [adds FileSpecifier to m_saves]
                                           [calls delete_surplus_saves()]

LOADING PHASE:
  [UI Dialog] --[load_quick_save_dialog()]--> [returns FileSpecifier]
              <--[user selects from m_saves]--

DELETION PHASE:
  [m_saves] --[delete_quick_save()]--> [unlink from filesystem]
                                       [erase from vector]

PRUNING PHASE:
  [m_saves] --[delete_surplus_saves(max)]--> [std::sort by operator<]
                                             [remove oldest N files]
                                             [erase from vector & disk]
```

## Learning Notes

1. **Metadata materialization:** Early Aleph One design pre-computes `formatted_time` and `formatted_ticks` during enumeration. Modern engines lazy-format in renderers, but this is cache-friendly when saves list is displayed repeatedly.

2. **Networking awareness:** The `saved_game_was_networked()` function signals that saves track multiplayer contextΓÇöimportant because a networked save (with peer latency, determinism windows) has different semantics than a single-player save.

3. **Singleton longevity:** Unlike many transient singletons, QuickSaves persists for the entire session. Its `m_saves` vector reflects the filesystem state at last enumerationΓÇöit must be re-enumerated if files are manipulated externally.

4. **Simplified player count tracking:** Storing `int16 players` directly in the struct suggests the engine needs to filter compatible saves (e.g., 4-player saves can't load into a 2-player game).

## Potential Issues

1. **Thread safety unclear:** The singleton's `instance()` and `m_saves` vector have no visible synchronization. If the network thread or audio thread accesses QuickSaves concurrently, data corruption could occur. Check `.cpp` for mutex guards.

2. **Enumeration contract ambiguous:** Does `enumerate()` clear old state or append? If appending, repeated enumerations accumulate duplicates. Modern practice would use `clear()` explicitly first.

3. **delete_surplus_saves semantics:** Does this mutate the vector in-place and unlink files from disk, or just unlink? If the latter, dangling `FileSpecifier` entries remainΓÇöunclear from header alone.

4. **No validation of `max_saves`:** Passing `max_saves=0` might delete all saves. Should have bounds checking or preconditions documented.

5. **Comparison operator only on time:** The `operator<` enables sorting, but doesn't define equality. If two saves have identical `save_time`, their relative order is unspecifiedΓÇöcould cause non-deterministic pruning.
