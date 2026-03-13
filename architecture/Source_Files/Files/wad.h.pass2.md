# Source_Files/Files/wad.h - Enhanced Analysis

## Architectural Role

WAD is the universal persistence and resource container for the entire Aleph One engine. It serves three critical roles: (1) **level storage**ΓÇöentire maps (geometry, objects, entities) are archived in WAD format; (2) **save game serialization**ΓÇöcomplete world state (player, monsters, platforms, dynamic effects) flows through WAD I/O via `game_wad.cpp`; (3) **mod/patch composition**ΓÇöparent-checksum tracking enables a mod DAG where derived WADs extend base WADs without duplication. This makes WAD the architectural *linchpin* between file I/O (FileHandler abstraction) and every major subsystem that needs persistent or transferable data.

## Key Cross-References

### Incoming (who depends on this file)

- **`game_wad.cpp/h`** ΓÇö Wraps WAD API for game lifecycle: `build_save_game_wad()`, `build_export_wad()`, level loading, player spawn restoration. Primary bridge from GameWorld to raw WAD I/O.
- **`preprocess_map_sdl.cpp`** ΓÇö Called during scenario loading to parse map metadata (`add_finishing_touches_to_save_file`), extract physics definitions, validate structure.
- **`wad_prefs.cpp`** ΓÇö Preferences persistence: reads/writes user settings via WAD serialization (leveraging flat format for reliability).
- **`import_definitions.cpp`** ΓÇö Physics unpacking: extracts monster/weapon/effect/projectile data from WAD tag streams.
- **Network subsystem** ΓÇö `network_games.cpp` distributes map checksums (`read_wad_file_checksum`) to validate synchronized level state across peers.
- **Rendering subsystem** ΓÇö Shape collection loading chains through WAD: shapes.cpp queries texture/bitmap metadata embedded in WAD data.
- **Sound subsystem** ΓÇö Audio resource lookup uses WAD tag extraction (`extract_type_from_wad`).
- **Lua/XML system** ΓÇö MML scenario parsing may reference WAD-resident resources; resource manager bridges to WAD queries.

### Outgoing (what this file depends on)

- **`tags.h`** ΓÇö Defines `WadDataType` constants (implicit dependency: `POINT_TAG`, `POLYGON_TAG`, `OBJECT_TAG`, etc.). No direct include in wad.h, but essential for understanding tag semantics in callers.
- **`FileHandler.h`** / `OpenedFile` ΓÇö Platform abstraction for file I/O (SDL_RWops wrapper). FileSpecifier and OpenedFile forward-declared; actual I/O delegated to FileHandler impl.
- **CSeries types** ΓÇö `uint32`, `int16`, `int8`, `byte` via implicit bundling (cstypes.h included via tags.h).
- **Checksum subsystem** (`crc.h/cpp`)** ΓÇö Implicitly called by `calculate_and_store_wadfile_checksum()` for CRC-32 computation.

## Design Patterns & Rationale

1. **Versioning via bit-flags** (`WADFILE_HAS_INFINITY_STUFF = 4`) ΓÇö Backward-compatible format evolution. Each version number is a cumulative capability mask; readers can detect features and branch. This is elegant for a 1990s archive format (avoids version number explosions).

2. **Flat data serialization** (`get_flat_data()` / `inflate_flat_data()`) ΓÇö WAD can serialize to contiguous byte stream for network transfer without per-tag overhead. Dual representation: (a) on-disk fragmented via directories, (b) in-memory inflated for access, (c) flattened for transit. Suggests latency-sensitive network code.

3. **Read-only mmap optimization** (`wad_data.read_only_data` pointer) ΓÇö If file data can be aliased directly (e.g., on platforms with safe memory protection), WAD loader avoids a copy. Memory-efficient for large levels on low-memory hardware (original Mac/PC era). Trade-off: file must remain open/locked.

4. **Inplace modification via offset tracking** (`entry_header.offset`, `append_data_to_wad(size_t offset)`) ΓÇö Supports patching: data can be appended at specific file offsets without re-serializing the entire WAD. Enables incremental saves (modifying specific level elements without rewriting).

5. **Directory-based random access** ΓÇö Directory entries map WAD index ΓåÆ file offset. Allows selective loading of individual WADs from a multi-WAD file without parsing header stream sequentially. Critical for fast map switching.

6. **Parent-checksum chain for mod composition** (`parent_checksum` field) ΓÇö A WAD can reference a "parent" by its CRC checksum. This enables mods as delta files: only override tags differ from base, others inherited. Elegant but requires checksums to uniquely identify versions (no content-addressing).

## Data Flow Through This File

**Loading a level:**
1. Player selects map ΓåÆ `open_wad_file_for_reading(FileSpecifier)` ΓåÆ returns `OpenedFile` handle
2. `read_wad_header(OpenedFile, wad_header*)` ΓåÆ parses 128-byte fixed header (version, checksum, dir offset, parent checksum)
3. `read_indexed_wad_from_file(OpenedFile, header, index, read_only=true)` ΓåÆ seeks to directory, reads entry_header stream, allocates `tag_data[]` array
4. For each tag: `extract_type_from_wad(wad_data, POLYGON_TAG, &len)` ΓåÆ **GameWorld receives raw polygon bytes** ΓåÆ `map.cpp` deserializes into `polygon` array
5. Repeat for objects, platforms, etc.
6. `free_wad()` deallocates if copied; if read-only, file stays open (implicit mmap lifetime)

**Saving a game:**
1. `create_wadfile(FileSpecifier, Type)` + `open_wad_file_for_writing()`
2. `create_empty_wad()` ΓåÆ allocate `wad_data` with zeroed tag count
3. **GameWorld serializes**: `append_data_to_wad(wad, PLAYER_TAG, &player, sizeof(player), 0)` repeatedly for all entity types
4. `write_wad(OpenedFile, header, wad, offset)` ΓåÆ writes flattened entry stream
5. `write_directorys()` ΓåÆ writes directory_entry array at header-specified offset
6. `calculate_and_store_wadfile_checksum()` ΓåÆ CRC whole file, store in header
7. `close_wad_file()` ΓåÆ flush and close

**Network synchronization:**
1. Before multiplayer: `read_wad_file_checksum(FileSpecifier)` ΓåÆ distribute to peers
2. Peer validation: each player's `read_wad_file_checksum(their_map_file)` must match broadcasted checksum
3. On mismatch: trigger map download or reject join (in `network_messages.cpp`)

## Learning Notes

- **Era-appropriate design:** WAD reflects 1990s constraints: no built-in compression, fixed-size directory (64 entries), contiguous flat-file layout. Modern engines use asset streaming, runtime LOD, and content-addressed storage.
- **Tag extensibility:** Unknown tags in older files are preservable (not parsed, just re-written). This is a lossy but resilient forward-compatibility pattern.
- **Deterministic serialization:** Big-endian or version-specific byte order (via `AStream.h`) ensures cross-platform saves work. Important for network sync.
- **Memory tier abstraction:** WAD exposes read-only aliasing to avoid copies when safe. This design is teaching: modern engines abstract memory hierarchy (VRAM, cache, page tables) more aggressively.
- **SetBetweenlevels() sentinel:** Prevents memory allocator interference during level load/unload transitions. Suggests level loading was a bottleneck or fragmentation issue.

## Potential Issues

1. **Fixed directory limit (64 entries)** ΓÇö A WAD file cannot exceed 64 logical data blocks. Large custom maps hitting this limit will fail to load silently or crash. No obvious expansion path without breaking format version 4 compatibility.

2. **Checksum collision vulnerability** ΓÇö Parent-checksum mod chain has no integrity proof. If two unrelated WADs happen to have the same CRC-32, mod loading silently uses wrong parent. CRC-32 has ~2^32 collision probability for large file sets; should use stronger hash.

3. **No per-tag versioning** ΓÇö If a POLYGON_TAG structure layout changes between engine versions, old save files cannot be migrated. Tag data format is opaque to WAD layer; incompatibility discovered at deserialization time, not WAD load time.

4. **Read-only mmap lifetime issue** ΓÇö If `read_only_data` alias is used and the source file is deleted/modified before `free_wad()` is called, undefined behavior results. No reference counting or file locking.

5. **Flat format overhead for small WADs** ΓÇö Serializing via `get_flat_data()` for network transfer inflates size (no per-tag compression). OK for few-KB objects; problematic for large textures or complex AI data.
