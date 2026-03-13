# Source_Files/RenderMain/shape_definitions.h

## File Purpose
Defines the in-memory layout and global storage for game shape collections (animated sprites, textures, scenery, enemies, etc.). Serves as the bridge between disk-resident collection files and engine rendering systems, using a 32-byte on-disk header format.

## Core Responsibilities
- Define `collection_header` structure for collection metadata and resource pointers
- Declare the global array of all loaded collection headers indexed by collection ID
- Establish layout contract for variable-format shape/shading data stored on disk or in memory
- Provide constants and structure sizes for disk I/O operations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `collection_header` | struct | On-disk/in-memory metadata for a single shape collection: flags, disk offsets for different shape formats, pointers to collection definition and shading lookup tables |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `collection_headers` | `struct collection_header[MAXIMUM_COLLECTIONS]` | static (file) | Global repository of all loaded collection headers, indexed by collection ID (0ΓÇô31) |

## Key Functions / Methods
None.

## Control Flow Notes
Not inferable from this file. This is a pure data definition header; initialization and lifecycle management are elsewhere (likely in resource loading/unloading code).

## External Dependencies
- **Includes:**
  - `shape_descriptors.h` ΓÇô collection IDs, shape descriptor bit-packing macros, and `MAXIMUM_COLLECTIONS` constant
- **Forward declares:**
  - `collection_definition` ΓÇô actual shape geometry/metadata (defined elsewhere)
- **Types used:**
  - `int16`, `uint16`, `int32`, `byte`, `std::vector<byte>` (from cstypes.h, indirectly)

## Notes
- Comment (Aug 2000) indicates a migration from MacOS handles to pointers for cross-platform compatibility.
- Disk size of `collection_header` is fixed at 32 bytes; actual in-memory size may differ due to padding or pointer sizes.
- `shading_tables` is a `std::vector`, meaning it's dynamically sized and allocated separately from the header struct.
