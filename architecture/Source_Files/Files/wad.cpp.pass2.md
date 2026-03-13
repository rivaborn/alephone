# Source_Files/Files/wad.cpp - Enhanced Analysis

## Architectural Role

WAD.cpp acts as the **serialization/deserialization bridge** between disk format and in-memory game data structures in Aleph One. It sits at the boundary between the Files subsystem and GameWorld/Rendering subsystems: all game resources (maps, textures, sprites, sounds) flow through wad.cpp during level load, and player state is serialized through it during save/network transmission. The file's version-evolution design (Marathon 1 Γåö Marathon Infinity compatibility) makes it a **backwards-compatibility adapter**, allowing the engine to ingest assets from multiple eras without format-specific branches at higher layers.

## Key Cross-References

### Incoming (who depends on this file)

**Direct callers visible in cross-ref:**
- `game_wad.cpp` ΓåÆ calls `read_indexed_wad_from_file()`, `append_data_to_wad()`, `remove_tag_from_wad()` to load/construct level and save-game WADs
- `placement.cpp` (GameWorld) ΓåÆ indirectly via game_wad.cpp for entity deserialization
- `wad_prefs.cpp` ΓåÆ uses WAD format for preferences persistence with corruption recovery
- `FileHandler.cpp` (Files subsystem) ΓåÆ maintains OpenedFile lifecycle that wad.cpp depends on

**Implicit consumers:**
- `Shape rendering pipeline` ΓåÆ shapes.cpp loads collections via file I/O, expects wad infrastructure to have validated checksums
- `Network subsystem` ΓåÆ `get_flat_data() / inflate_flat_data()` for multiplayer level transfer
- `XML config system` ΓåÆ levels defined in XML trigger wad file discovery and loading

### Outgoing (what this file depends on)

**Within Files subsystem:**
- `FileHandler.h/cpp` ΓåÆ OpenedFile abstraction for cross-platform I/O (SDL_RWops wrapper)
- `Packing.h` ΓåÆ Stream-based pack/unpack helpers for binary struct serialization
- `crc.h` ΓåÆ CRC32 calculation for file integrity validation

**External subsystems:**
- `game_errors.h` ΓåÆ `set_game_error()` for error reporting (not fatal; allows graceful degradation)
- `tags.h` ΓåÆ Constants for WAD tag types (POINT_TAG, MAP_INFO_TAG, etc.ΓÇöthe extensibility mechanism)
- `cseries.h` ΓåÆ Platform abstraction: fixed-width integers, memory macros, assertion helpers
- **Global function `level_transition_malloc()`** (defined elsewhere in Game World) ΓåÆ Specialized allocator for between-levels data; signals that wad allocation is coordinated with level lifecycle

## Design Patterns & Rationale

**Version Envelope Pattern**: Each WAD is wrapped with a versioned header (v0ΓÇôv4) that declares both file format version and data version. Entry headers and directory entries exist in "old" and "new" forms (Marathon 1 vs. later). At read time, the code uses the header version to dispatch to the correct unpacking routine (`unpack_old_entry_header` vs. `unpack_entry_header`). This avoids having multiple wad readers; instead, **translation happens at the boundary** (comments from 2000ΓÇô2001 show careful iterations on this).

**Tag-Based Extensibility**: The in-memory `wad_data` structure is an array of `tag_data` records (type, pointer, length). New asset types can be added without changing WAD formatΓÇöjust define a new tag constant and the append/extract machinery handles it generically. This design enabled the engine to grow from Marathon 1 through Infinity without format incompatibility.

**Lazy Read, Eager Write**:
- **Reading**: Raw WAD is read into a temporary buffer, then converted to tagged `wad_data` in-memory representation; the raw buffer is freed for read-only WADs (memory pressure in the 1990s made this important).
- **Writing**: `append_data_to_wad()` is called repeatedly to build the WAD, then `write_wad()` flushes all tags to disk in a single pass. The caller is responsible for setting up the header and checksumming.

**Checksum Validation**: `calculate_and_store_wadfile_checksum()` computes CRC32 over the entire file *except* the checksum field itself (zeroed during computation). This allows networked clients to verify that received WADs haven't been corrupted or tampered withΓÇöcritical for peer-to-peer multiplayer integrity.

**Separate Read-Only vs. Modifiable Paths**: `convert_wad_from_raw()` vs. `convert_wad_from_raw_modifiable()` reflect two use cases:
- Read-only: Level WADs are loaded once per level (memory-mapped backing buffer kept)
- Modifiable: Save games and dynamically-created WADs can be edited and re-serialized

## Data Flow Through This File

1. **Load Time (Level Start)**:
   ```
   File on Disk (binary WAD)
      ΓåÆ open_wad_file_for_reading(FileSpecifier, OpenedFile)
      ΓåÆ read_wad_header() [validates version/data_version/wad_count]
      ΓåÆ read_indexed_wad_from_file(index, read_only=true)
         ΓåÆ size_of_indexed_wad() [scan directory to find raw length]
         ΓåÆ read_indexed_wad_from_file_into_buffer() [load raw bytes]
         ΓåÆ convert_wad_from_raw() [parse entry headers, build tag array]
      ΓåÆ returns: wad_data with tag array in memory (raw buffer kept for read-only access)
   ```
   **Key state**: Directory offsets computed by `calculate_directory_offset()` using header metadata. Raw buffer is "read-only" and should not be freed.

2. **Save Time (Save Game)**:
   ```
   In-Memory wad_data (tags)
      ΓåÆ create_empty_wad()
      ΓåÆ append_data_to_wad(type=MAP_TAG, data=..., size=...) [repeated]
      ΓåÆ append_data_to_wad(type=PLAYER_TAG, ...)
      ΓåÆ ... (dozens of tag types)
      ΓåÆ open_wad_file_for_writing(FileSpecifier, OpenedFile)
      ΓåÆ fill_default_wad_header()
      ΓåÆ write_wad(wad, offset=SIZEOF_wad_header)
      ΓåÆ write_directorys() [write directory entries after all WADs]
      ΓåÆ write_wad_header()
      ΓåÆ calculate_and_store_wadfile_checksum()
      ΓåÆ close_wad_file()
   ```
   **Key insight**: Directory is written *after* WADs, so offsets must be calculated in advance.

3. **Network Transfer**:
   ```
   wad_data ΓåÆ get_flat_data() ΓåÆ opaque blob (magic + length + header + raw) ΓåÆ network
   network ΓåÆ inflate_flat_data() ΓåÆ wad_data
   ```

## Learning Notes

**For studying this engine**, wad.cpp teaches several lessons:

1. **Format Versioning at Scale**: Dating from 1994, the WAD format has survived 7+ years of evolution (comments through 2002) by versioning both the file structure and the data schema independently. This is more robust than many contemporary engines.

2. **Binary Serialization Discipline**: The use of `Packing.h` helpers (`pack_wad_header()`, `unpack_entry_header()`) centralizes byte-order conversion, making ports to big-endian machines straightforward. (Compare: many engines hardcoded little-endian assumptions.)

3. **Memory Accounting**: The `BetweenLevels` flag and level-transition allocator integration show deliberate design for the 1990s Mac memory model (fragmentation, heap pressure). Modern engines would use VRAM streaming, but this is appropriate for its era.

4. **Checksum Before Network**: Validating file integrity *before* sending over the network is a lightweight form of anticheat. Marathon's peer-to-peer multiplayer needed this to prevent corrupted WADs from crashing the session.

**What's idiomatic to this era**:
- No RAII; explicit `malloc` / `free` with nullchecks
- Packed binary structures with manual byte-order management
- Comment-heavy change history (pre-git version control)
- No compression or streaming; entire level must fit in RAM

Modern engines often use:
- Asset streaming and virtual memory to avoid 2├ù memory overhead
- Delta compression for network transfer
- Lazy deserialization (parse on-demand, not at load)
- Content-addressed storage (content hash as name, not versioned file format)

## Potential Issues

1. **2├ù Memory Overhead During Load** (line comment): A level WAD is read into a temporary buffer, then parsed into `wad_data` structures, keeping both in RAM during load. For large levels, this could be optimized by parsing on-the-fly or using memory-mapped I/O. This is mitigated by `level_transition_malloc()`, which is a memory pool separate from the gameplay heap.

2. **Directory Offset Calculation Fragility**: `calculate_directory_offset()` computes file offsets by summing entry sizes; off-by-one errors in header sizing can cause misalignment. The code uses `assert()` and static size constants (`SIZEOF_entry_header`, `SIZEOF_directory_entry`) to guard this, but there's no runtime validation that the directory offset matches the actual file position after all WADs are written.

3. **Implicit Checksum State**: `calculate_and_store_wadfile_checksum()` zeroes the checksum field, computes CRC, writes header, then re-writes header with CRC. If an error occurs between the two writes, the file is left in an inconsistent state (checksum=0 but file has changed). No rollback or atomic write strategy is used.

4. **Linear Tag Search**: `extract_type_from_wad()` is O(n) in tag count. For WADs with dozens of tag types, this is acceptable, but there's no index or hash table optimization. This is rarely a bottleneck (tags are read at load time, not per-frame).

5. **Version Checking Too Strict**: Header validation checks `version > CURRENT_WADFILE_VERSION`, which rejects future file formats outright. A more lenient approach (skip unknown tags gracefully) would allow forward compatibility, but would require more complex parsing logic.
