# Extras/physics_patches.cpp - Enhanced Analysis

## Architectural Role

This is a **build-time content-creation utility**, not a runtime engine component. It belongs to Aleph One's offline "physics patching pipeline"ΓÇöalongside tools like `tools/dumpwad.cpp` and the `Extras/extract/` family. Its purpose is to enable efficient distribution of physics data updates (new weapon definitions, monster behaviors, etc.) by computing and encoding only the byte-level differences between an original and new physics WAD, then storing those deltas in a patch WAD linked to the original via checksum. This allows patch files to be minimal and validate cleanly at runtime (a player can apply patch P only if their base physics file's checksum matches the parent checksum stored in P).

## Key Cross-References

### Incoming (who depends on this file)
- **Nobody in the engine runtime.** This is a standalone command-line tool, invoked manually during content authoring.
- Referenced indirectly through the build/content workflow: developers use it to generate `.patch` files that end up in distribution packages.

### Outgoing (what this file depends on)
- **Files subsystem heavily** (`wad.h`): `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `extract_type_from_wad()`, `create_empty_wad()`, `append_data_to_wad()`, `write_wad_header()`, `write_wad()`, `write_directorys()`, `calculate_and_store_wadfile_checksum()`, `calculate_wad_length()`.
- **CSeries abstractions** (`macintosh_cseries.h`, `BStream`-style operations): `c2pstr()`, `strcpy()`, `fprintf()`, `malloc()`, Mac-style `FSSpec` structs and `FSMakeFSSpec()`.
- **Definition headers** (`extensions.h`, `tags.h`): Populates the global `definitions[]` array and `NUMBER_OF_DEFINITIONS` macro.
- **Data structures** (`map.h`, `projectiles.h`, `weapons.h`, etc.): Only for header inclusion; the actual binary data comes from the WAD files, not these compile-time declarations.

## Design Patterns & Rationale

**Binary Delta Encoding (Byte-Level)**  
Rather than storing full physics WAD files in patches, the tool performs byte-by-byte comparison and records only contiguous non-matching regions (offset + length + new bytes). This is era-appropriateΓÇöpredates tools like `bsdiff` and `xdelta`, and leverages the WAD format's already-structured tag system to organize deltas by definition type.

**WAD Format as Universal Serialization**  
The patch file is itself a WAD (same format, validated by `read_wad_header()`). This is elegant: the same deserialization pipeline that loads game state and physics definitions also loads patches. No need for a separate binary patch format. The `write_patchfile()` function creates a WAD header, appends delta data, and writes a directoryΓÇöidentical structure to any other WAD.

**Parent Checksum Validation**  
The patch stores `parent_checksum` (the original file's checksum) in the WAD header. This enforces correctness: at runtime, a player's engine can refuse to apply a patch if their base physics file's checksum doesn't match. Without this, applying patch P to the wrong version of the base file silently produces corrupted data.

**Strict Assertion on Structure**  
`assert(original_data && new_data && new_length==original_length)` indicates the tool assumes both physics files have identical *structure* (same definitions, same sizes per tag). This is reasonable for a build tool (physics evolution is controlled, not arbitrary), but it means the tool is fragile: mismatched versions fail hard instead of gracefully reporting differences.

**Diagnostic Logging to stderr**  
Heavy use of `fprintf(stderr, ...)` throughout suggests this tool was built for interactive use or CI/batch processing, where developers want to see per-definition diffs (tag, offset, length). This is not user-friendly output; it's engineer-facing debugging output.

## Data Flow Through This File

```
Input Files (command-line args)
    Γåô
FSMakeFSSpec() ΓåÆ FileDesc structures
    Γåô
create_delta_wad_file()
    Γö£ΓöÇΓåÆ get_physics_wad_from_file(original)
    Γöé   Γö£ΓöÇ open_wad_file_for_reading()
    Γöé   Γö£ΓöÇ read_wad_header() ΓåÆ retrieve parent_checksum
    Γöé   Γö£ΓöÇ read_indexed_wad_from_file() ΓåÆ in-memory wad_data
    Γöé   ΓööΓöÇ close_wad_file()
    Γöé
    Γö£ΓöÇΓåÆ get_physics_wad_from_file(new)
    Γöé   ΓööΓöÇ [same as above, discard checksum]
    Γöé
    Γö£ΓöÇΓåÆ Loop through NUMBER_OF_DEFINITIONS
    Γöé   For each definition tag:
    Γöé   Γö£ΓöÇ extract_type_from_wad(original) ΓåÆ original_data buffer
    Γöé   Γö£ΓöÇ extract_type_from_wad(new) ΓåÆ new_data buffer
    Γöé   Γö£ΓöÇ Byte-by-byte memcmp loop ΓåÆ find first_nonmatching_offset, nonmatching_length
    Γöé   Γö£ΓöÇ fprintf(stderr, "Tag: %.4s...")  [diagnostic]
    Γöé   ΓööΓöÇ append_data_to_wad(delta_wad, tag, &new_data[offset], length, offset)
    Γöé
    Γö£ΓöÇΓåÆ write_patchfile(delta_wad, parent_checksum)
    Γöé   Γö£ΓöÇ create_wadfile() ΓåÆ new output file
    Γöé   Γö£ΓöÇ fill_default_wad_header() ΓåÆ populate header struct
    Γöé   Γö£ΓöÇ write_wad_header() [first write, before data]
    Γöé   Γö£ΓöÇ write_wad() ΓåÆ write delta data
    Γöé   Γö£ΓöÇ Update header with directory offset / wad_count
    Γöé   Γö£ΓöÇ write_wad_header() [second write, after data]
    Γöé   Γö£ΓöÇ write_directorys()
    Γöé   Γö£ΓöÇ calculate_and_store_wadfile_checksum()
    Γöé   ΓööΓöÇ close_wad_file()
    Γöé
    ΓööΓöÇΓåÆ free_wad() ΓåÆ cleanup
    
Output: Patch WAD file (with parent_checksum metadata)
```

**State Transitions**: The tool has no persistent state; it's a pure pipeline (parse args ΓåÆ read files ΓåÆ compare ΓåÆ write ΓåÆ exit). The only "state" is in-memory buffers.

## Learning Notes

**What a Developer Studying This Learns**:
1. **The WAD format is foundational**: Used for game saves, resource packing, and even patch distribution. The engine assumes all data is WAD-packaged.
2. **Tag-based organization enables modularity**: Physics data is keyed by `tag` (e.g., weapon definitions, monster definitions), and the comparison loop is generic over all tags. Adding a new definition type doesn't require changes to the patching logic.
3. **Checksums are critical for validation**: Prevents silent corruption from version mismatches. Runtime engines probably validate `parent_checksum` before applying a patch.
4. **Era-appropriate engineering**: Hand-rolled byte-level delta (1995 era) vs. modern binary diff libraries. The simplicity is intentional; there's no need for sophisticated delta algorithms when the data is structured and versions are controlled.

**Idiomatic to Aleph One / Marathon Era**:
- Mac OS 68K/PowerPC heritage: `FSSpec`, `c2pstr()`, `FileDesc`, big-endian assumptions.
- Resource-centric data model: Everything is a WAD; versioning is done via `data_version` fields in headers.
- Offline tooling as first-class: The codebase includes many build-time utilities (extract sounds, dump WAD, inspect resources). Data authoring is not separate from code.
- Deterministic, validated serialization: Checksums, parent linkage, strict structure assertions ensure reproducibility and prevent corruption.

## Potential Issues

1. **Unused Function**: `level_transition_malloc()` is dead codeΓÇödefined but never called. Likely legacy from refactoring or copy-paste from another tool.

2. **Strict Assertion, No Graceful Error Handling**: The assertion `assert(original_data && new_data && new_length==original_length)` will crash if:
   - Definition structure changed between versions (e.g., weapon definitions grew in size).
   - One file is missing a definition tag the other has.
   
   For a build tool, this is acceptable (catch errors early), but a production tool might want to report differences and skip mismatched definitions.

3. **No Patch Format Versioning**: The patch WAD format itself (how delta regions are encoded) has no version field. If the format ever needs to evolve, old tools cannot distinguish old/new patch formats. This is mitigated by the `PATCH_FILE_TYPE` constant, but future-proofing would add a format version in the WAD header.

4. **Binary-Only Output**: Patches are binary blobs with no human-readable representation. Debugging a corrupted patch requires manual hexdump inspection. Modern tools often emit human-readable diffs (JSON, unified diff format, etc.).

5. **No Validation of Delta Application**: The tool *creates* patches but doesn't validate that they can be applied. A reciprocal tool that applies patches and verifies the result would catch authoring errors.
