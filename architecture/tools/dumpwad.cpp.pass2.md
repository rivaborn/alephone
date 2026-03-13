# tools/dumpwad.cpp - Enhanced Analysis

## Architectural Role

This is a standalone offline inspection utility for Marathon WAD archives, operating outside the game engine proper. It serves the dual purpose of debugging tool and educational reference for the WAD format structure. Unlike the game's runtime WAD loader, it prioritizes human-readable output over performance, unpacking all metadata and tag information sequentially. The tool demonstrates how the engine's binary serialization layer (Packing.cpp, AStream) is used for reverse-compatible version handlingΓÇöspecifically supporting both old and new directory entry formats in a single utility.

---

## Key Cross-References

### Incoming (who depends on this file)
- **None.** This is a standalone CLI utility with no callers within the engine.
- Users invoke it directly from the command line for offline WAD inspection.

### Outgoing (what this file depends on)
- **wad.cpp** ΓåÆ `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()`, `read_indexed_wad_from_file()`, `get_indexed_directory_data()`, `calculate_directory_offset()`, `free_wad()`
- **FileHandler_SDL.cpp** ΓåÆ `FileSpecifier`, `OpenedFile`, `read_from_file()`
- **Packing.cpp** ΓåÆ `StreamToValue()`, `StreamToBytes()` (binary deserialization macros)
- **map.h** ΓåÆ `directory_entry`, `old_directory_entry`, `directory_data`, `wad_header`, `wad_data`, `tag_data`
- **crc.cpp** (included, not directly called)
- **Logging.cpp, XML_ElementParser.cpp, resource_manager.cpp** (included but stubbed out)

---

## Design Patterns & Rationale

**1. Direct .cpp Inclusion for Tool Compilation**
The tool includes implementation files (`wad.cpp`, `FileHandler_SDL.cpp`, etc.) rather than linking against pre-compiled objects. This is a deliberate choice to minimize build complexityΓÇöthe tool compiles in isolation without needing the full engine build infrastructure. This pattern trades compilation time for deployment simplicity (single-file or minimal-file tool).

**2. Stub Function Pattern**
Game-required functions (`set_game_error`, `dprintf`, `alert_user`, `level_transition_malloc`) are stubbed with empty implementations to satisfy linker dependencies from included .cpp files without requiring the full engine. This is a lightweight alternative to conditional compilation, allowing code reuse without heavyweight conditional logic.

**3. Version Compatibility Via Binary Layout Branching**
The tool reads directory entries in two different formats (`SIZEOF_old_directory_entry` vs. `SIZEOF_directory_entry`) and branches on the actual size to deserialize correctly. This is simpler than a versioned factory patternΓÇöthe binary format itself indicates which path to take. The design assumes the header's `entry_header_size` field reliably indicates the directory entry schema.

**4. Linear, Imperative Flow**
Strictly sequential processing: validate args ΓåÆ open file ΓåÆ read header ΓåÆ iterate directory ΓåÆ dump metadata. No dynamic state, no callbacks, no frame-based updates. This is typical for offline inspection tools that prioritize readability over concurrency.

---

## Data Flow Through This File

```
Command-line args (WAD file path)
    Γåô
open_wad_file_for_reading() [wad.cpp]
    Γåô
read_wad_header() ΓåÆ wad_header struct (version, checksums, counts, offsets)
    Γåô
Print header metadata to stdout
    Γåô
read_directory_data() ΓåÆ raw directory metadata block
    Γåô
For each of wad_count entries:
    Γö£ΓöÇ calculate_directory_offset() ΓåÆ file offset
    Γö£ΓöÇ read_from_file() ΓåÆ raw directory entry bytes
    Γö£ΓöÇ unpack_directory_entry() or unpack_old_directory_entry() (version dispatch)
    Γö£ΓöÇ get_indexed_directory_data() ΓåÆ mission/environment/entry-point flags + level name
    Γö£ΓöÇ unpack_directory_data() ΓåÆ deserialize StreamToValue/StreamToBytes
    Γö£ΓöÇ Decode and print mission flags (extermination, exploration, retrieval, repair, rescue)
    Γö£ΓöÇ Decode and print environment flags (vacuum, magnetic, rebellion, low gravity)
    Γö£ΓöÇ Decode and print entry point flags (single player, coop, carnage, CTF, KoH, defense, rugby)
    Γö£ΓöÇ read_indexed_wad_from_file() ΓåÆ wad_data (loaded resource with tags)
    ΓööΓöÇ Enumerate and print all tag_data entries (tag ID, length)
    Γåô
free_wad() ΓåÆ cleanup
    Γåô
stdout (human-readable WAD manifest)
```

The tool never modifies the WADΓÇöit's purely read-only inspection.

---

## Learning Notes

**What this file teaches about Aleph One:**

1. **WAD Format Layers:**
   - Header (metadata, version, checksums, directory offset)
   - Directory data (per-level: mission type, environment hazards, entry points, level name)
   - Directory entries (per-entry: file offset, length, index)
   - Tag data (per-tag: 4-char FourCC tag ID, data length, pointer)

2. **Binary Deserialization Pattern:**
   The `unpack_directory_data()` function is a minimal example of the engine's serialization layer:
   ```cpp
   StreamToValue(S, field);  // Byte-swapped big-endian to native
   StreamToBytes(S, array, length);  // Raw byte copy
   ```
   This is how the engine avoids floating-point math in serializationΓÇöfixed-width integer types and big-endian network byte order.

3. **Flag-Based Metadata Encoding:**
   Marathon uses bitflags extensively (mission_flags, environment_flags, entry_point_flags) rather than enums or tagged unions. This saves space in on-disk formats and allows composable mission definitions.

4. **Version Compatibility Pattern:**
   The tool handles both old and new directory entry layouts by branching on byte size. The game likely does the same when loading maps, suggesting a "soft" versioning strategy: binary layout evolves, but old layouts remain readable.

5. **Idiomatic 2000s Engine Practices:**
   - Direct pointer arithmetic and buffer management (no RAII)
   - Command-line utilities as standalone compiled units
   - Human-readable flag decoding in tools (not in the engine proper, which likely stores raw uint32 values)

---

## Potential Issues

1. **Checksum Validation Not Performed:**
   The tool reads `h.checksum` and `h.parent_checksum` but never validates them against calculated CRC. A corrupted WAD would not be detected until tag parsing failsΓÇölate discovery.

2. **No Bounds Validation on Directory Entry Size:**
   ```cpp
   if (base_entry_size > SIZEOF_directory_entry)
       base_entry_size = SIZEOF_directory_entry;
   ```
   This silently truncates if the header claims a larger size, but doesn't validate that `base_entry_size >= SIZEOF_old_directory_entry`. A malformed header could cause unpack_old_directory_entry() or unpack_directory_entry() to read uninitialized memory.

3. **Implicit Size Assumptions:**
   The condition `if (h.application_specific_directory_data_size == SIZEOF_directory_data)` only decodes level metadata if it matches exactly. Non-standard data sizes are silently skippedΓÇöcould mask format corruption.

4. **No Loop Bounds on Tag Enumeration:**
   ```cpp
   for (int t=0; t<w->tag_count; t++)
       printf("    '%c%c%c%c' ...", w->tag_data[t].tag, ...);
   ```
   Assumes `w->tag_data` is allocated with at least `tag_count` elements. If `read_indexed_wad_from_file()` fails to allocate or populate correctly, out-of-bounds access is silent.

5. **Error Recovery is Minimal:**
   Early `exit(1)` on any error prevents incremental inspection. A tool that could skip bad entries and continue would be more robust for forensics on partially corrupted WADs.
