# Source_Files/Files/wad.h

## File Purpose
Defines the WAD (archive) file format structures and I/O functions for the Marathon/Aleph One game engine. WAD files serve as containers for game data (levels, assets, etc.), with version control and checksum integrity support. Provides both low-level file I/O and higher-level WAD data manipulation APIs.

## Core Responsibilities
- Define binary WAD file format (header, directory entries, data headers) with version evolution
- Read WAD files from disk and parse internal structures (headers, directories, tag data)
- Write and create WAD files with proper checksums and metadata
- Extract typed data from loaded WAD structures in memory
- Manage parent-child WAD relationships via checksums
- Provide flat serialization for data transfer and inflation back to in-memory form
- Memory lifecycle management for loaded WAD data

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `wad_header` | struct | File-level header (128 bytes fixed); stores version, checksum, directory offset, parent tracking |
| `directory_entry` | struct | Maps WAD index to file offset/length; minimal 10 bytes; supports inplace WAD modification |
| `entry_header` | struct | Header for individual data entries (tags) within a WAD; 16 bytes; tracks offset for expansion |
| `tag_data` | struct | In-memory representation of a single typed data block; holds pointer, length, type tag, patch offset |
| `wad_data` | struct | Complete in-memory WAD; array of `tag_data`, read-only flag, tag count |
| `WadDataType` | typedef | `uint32` tag identifier for data types (e.g., `POINT_TAG`, `POLYGON_TAG` from tags.h) |

## Global / File-Static State
None.

## Key Functions / Methods

### read_indexed_wad_from_file
- Signature: `struct wad_data *read_indexed_wad_from_file(OpenedFile& OFile, struct wad_header *header, short index, bool read_only)`
- Purpose: Load a single WAD from a multi-WAD file and deserialize its entries into memory
- Inputs: Open file handle, parsed header, WAD index, read-only flag
- Outputs/Return: Allocated `wad_data` structure with loaded entries
- Side effects: Allocates memory; may set read-only pointer to mmap'd file data
- Calls: Not inferable
- Notes: Called during level/asset loading; read-only mode optimizes memory by aliasing file data

### extract_type_from_wad
- Signature: `void *extract_type_from_wad(struct wad_data *wad, WadDataType type, size_t *length)`
- Purpose: Retrieve a single tagged data block from a loaded WAD
- Inputs: WAD, tag identifier, pointer to length output
- Outputs/Return: Pointer to tag data; length written to `*length`
- Side effects: None
- Calls: Not inferable
- Notes: Core API for extracting level geometry, objects, etc.

### write_wad
- Signature: `bool write_wad(OpenedFile& OFile, struct wad_header *file_header, struct wad_data *wad, int32 offset)`
- Purpose: Serialize a WAD in memory to disk at specified offset
- Inputs: Open file, header, in-memory WAD, file offset
- Outputs/Return: Success flag
- Side effects: Writes to file; updates header metadata
- Calls: Not inferable
- Notes: Used during save and WAD creation

### append_data_to_wad
- Signature: `struct wad_data *append_data_to_wad(struct wad_data *wad, WadDataType type, const void *data, size_t size, size_t offset)`
- Purpose: Add or modify a tagged data entry in an in-memory WAD
- Inputs: WAD, tag type, data pointer, size, patch offset
- Outputs/Return: Modified WAD (may be reallocated)
- Side effects: Allocates/reallocates memory
- Calls: Not inferable
- Notes: Supports inplace patches via `offset` parameter

### free_wad
- Signature: `void free_wad(struct wad_data *wad)`
- Purpose: Deallocate an in-memory WAD and all its tag data
- Inputs: WAD pointer
- Outputs/Return: None
- Side effects: Frees memory; does not close file if data was read-only mmap
- Calls: Not inferable

### calculate_wad_length
- Signature: `int32 calculate_wad_length(struct wad_header *file_header, struct wad_data *wad)`
- Purpose: Compute serialized size of a WAD for disk write
- Inputs: File header (version info), WAD
- Outputs/Return: Total byte length
- Side effects: None
- Calls: Not inferable

**Notes on other functions:**
- Checksum functions: `wad_file_has_checksum`, `read_wad_file_checksum`, `calculate_and_store_wadfile_checksum` ΓÇö integrity validation and parent-chain tracking
- Directory functions: `read_directory_data`, `get_indexed_directory_data`, `set_indexed_directory_offset_and_length` ΓÇö internal directory management
- Flat data: `get_flat_data`, `inflate_flat_data` ΓÇö serialization API for network/save transfers
- Utility: `create_empty_wad`, `fill_default_wad_header`, `dump_wad`

## Control Flow Notes
- **Init/Load**: `create_wadfile` ΓåÆ `open_wad_file_for_reading` ΓåÆ `read_wad_header` ΓåÆ `read_indexed_wad_from_file` ΓåÆ extract game data via `extract_type_from_wad`
- **Save/Write**: `create_wadfile` ΓåÆ `open_wad_file_for_writing` ΓåÆ `append_data_to_wad` (build WAD) ΓåÆ `write_wad` ΓåÆ `calculate_and_store_wadfile_checksum` ΓåÆ `close_wad_file`
- **Shutdown**: `free_wad` ΓåÆ `close_wad_file`
- Supports multi-WAD files; directory offset and count tracked in header; checksums enable parent-child (mod/patch) relationships

## External Dependencies
- `#include "tags.h"` ΓÇö WAD data type constants (`POINT_TAG`, `POLYGON_TAG`, `OBJECT_TAG`, etc.) and file typecode enums
- Forward declared: `FileSpecifier`, `OpenedFile` ΓÇö file abstraction layer (defined elsewhere, likely platform-specific)
- Standard: `uint32`, `int32`, `int16`, `byte` ΓÇö defined in `cstypes.h` (bundled with tags.h)
- Uses version constants `WADFILE_HAS_INFINITY_STUFF`, etc. for format evolution tracking
