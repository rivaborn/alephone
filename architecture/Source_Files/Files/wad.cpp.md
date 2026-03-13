# Source_Files/Files/wad.cpp

## File Purpose
Implementation of the WAD (Where's All the Data) file format handler for the Aleph One game engine. Manages reading, writing, and in-memory representation of compressed/structured game data containers that store maps, textures, sounds, and other resources. Handles version compatibility from Marathon 1 through Marathon Infinity.

## Core Responsibilities
- Read/write WAD files with multi-version format support (versions 0ΓÇô4)
- Parse binary WAD headers and directory entries; handle endianness conversion
- Load raw WAD data into memory and convert to tagged internal structures
- Support both read-only (memory-mapped) and modifiable (copied) WAD representations
- Manage tag-based data extraction and manipulation (append/remove operations)
- Calculate and validate CRC checksums for file integrity
- Allocate memory via standard `malloc` or level-transition allocator based on context
- Pack/unpack binary structures with format versioning (old vs. new entry headers/directory entries)
- Support flat (serialized) WAD transfer for network/IPC scenarios

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `wad_header` | struct | File header: version, checksum, directory offset, WAD count, entry/directory sizes |
| `directory_entry` / `old_directory_entry` | struct | Disk format: offset, length, optional index for overlays (v2+) |
| `entry_header` / `old_entry_header` | struct | Per-tag metadata: tag type, next offset (linked list), data length, optional offset field |
| `tag_data` | struct | In-memory tag: type, data pointer, length, patch offset |
| `wad_data` | struct | Loaded WAD: tag count, tag array, optional read-only backing buffer |
| `OpenedFile` | class (external) | File handle abstraction for I/O |
| `FileSpecifier` | class (external) | File path abstraction and utilities |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `internal_data` | `wad_internal_data*[3]` | static | Parallel tracking of open WAD file internal structures (up to 3 concurrent files) |
| `BetweenLevels` | `bool` | static | Controls allocation strategy: `true` uses level-transition allocator; `false` uses standard `malloc` |

## Key Functions / Methods

### read_wad_header
- **Signature:** `bool read_wad_header(OpenedFile& OFile, wad_header *header)`
- **Purpose:** Read and deserialize the file header from disk
- **Inputs:** Open file handle, header struct pointer
- **Outputs/Return:** `true` on success; populates `header`; sets game error on failure
- **Side effects:** Reads 128 bytes at offset 0; validates version/data_version/wad_count
- **Calls:** `read_from_file()`, `unpack_wad_header()`, `set_game_error()`
- **Notes:** Rejects versions > CURRENT_WADFILE_VERSION or data_version > 2; requires wad_count ΓëÑ 1

### read_indexed_wad_from_file
- **Signature:** `wad_data *read_indexed_wad_from_file(OpenedFile& OFile, wad_header *header, short index, bool read_only)`
- **Purpose:** Load a single WAD entry (by index) from disk into memory, converting to internal representation
- **Inputs:** Open file, header, WAD index, read-only flag
- **Outputs/Return:** Pointer to `wad_data` struct; `NULL` on error
- **Side effects:** Allocates memory (via `level_transition_malloc` or `malloc`); reads raw WAD; may free raw buffer if modifiable
- **Calls:** `size_of_indexed_wad()`, `read_indexed_wad_from_file_into_buffer()`, `convert_wad_from_raw()` or `convert_wad_from_raw_modifiable()`, `set_game_error()`
- **Notes:** Pads buffer by (SIZEOF_entry_header - SIZEOF_old_entry_header) for Marathon 1 compatibility; allocates 2├ù level size per comments

### extract_type_from_wad
- **Signature:** `void *extract_type_from_wad(wad_data *wad, WadDataType type, size_t *length)`
- **Purpose:** Linear search for and return a tagged data block from an in-memory WAD
- **Inputs:** Loaded WAD, tag type, length pointer
- **Outputs/Return:** Pointer to tag data; `NULL` if not found; writes tag length to `*length`
- **Side effects:** None (read-only)
- **Calls:** None
- **Notes:** Tag search is O(n); length set to 0 if tag not found

### write_wad
- **Signature:** `bool write_wad(OpenedFile& OFile, wad_header *file_header, wad_data *wad, int32 offset)`
- **Purpose:** Write all tags from an in-memory WAD to disk, serializing entry headers and data sequentially
- **Inputs:** Open file, header (for version/entry-header-size), WAD, file offset
- **Outputs/Return:** `true` on success
- **Side effects:** Writes entry_header + data sequentially to file; maintains running offset; sets game error on I/O failure
- **Calls:** `get_entry_header_length()`, `pack_old_entry_header()` or `pack_entry_header()`, `write_to_file()`
- **Notes:** Builds linked-list offsets (next_offset); last tag has next_offset=0; data must be modifiable (asserts !read_only_data)

### calculate_and_store_wadfile_checksum
- **Signature:** `void calculate_and_store_wadfile_checksum(OpenedFile& OFile)`
- **Purpose:** Compute CRC32 checksum of entire file and store in header
- **Inputs:** Open file (for read/write)
- **Outputs/Return:** None
- **Side effects:** Zeroes header.checksum, writes header twice (before/after CRC), updates file
- **Calls:** `read_wad_header()`, `write_wad_header()`, `calculate_crc_for_opened_file()`
- **Notes:** Ensures checksum field is not included in CRC computation

### append_data_to_wad
- **Signature:** `wad_data *append_data_to_wad(wad_data *wad, WadDataType type, const void *data, size_t size, size_t offset)`
- **Purpose:** Add or replace a tagged data block in a modifiable WAD; grows tag array if needed
- **Inputs:** WAD, tag type, data pointer, data size, patch offset
- **Outputs/Return:** Updated WAD pointer (same or reallocated)
- **Side effects:** Allocates/reallocates tag_data array; copies data; frees old data if tag exists
- **Calls:** `malloc()`, `memcpy()`, `alert_out_of_memory()`, `objlist_copy()`, `free()`
- **Notes:** Asserts non-zero size and !read_only_data; size=0 no longer allowed per comments

### remove_tag_from_wad
- **Signature:** `void remove_tag_from_wad(wad_data *wad, WadDataType type)`
- **Purpose:** Remove a tagged data block from a modifiable WAD, shrinking tag array
- **Inputs:** WAD, tag type
- **Outputs/Return:** None
- **Side effects:** Reallocates tag_data array; compacts tag list; calls alert_out_of_memory on allocation failure
- **Calls:** `malloc()`, `objlist_copy()`, `free()`, `alert_out_of_memory()`
- **Notes:** Does nothing if tag not found; asserts !read_only_data

### get_flat_data / inflate_flat_data
- **Signature:** 
  - `void *get_flat_data(FileSpecifier& File, bool use_union, short wad_index)`
  - `wad_data *inflate_flat_data(void *data, wad_header *header)`
- **Purpose:** Serialize a single WAD to a self-contained opaque binary blob (for network transfer); deserialize it back
- **Inputs:** File spec + index (get); opaque data pointer (inflate)
- **Outputs/Return:** Blob with encapsulated header and raw WAD data (get); loaded WAD struct (inflate)
- **Side effects:** Allocates memory for blob; validates magic cookie and length on inflate
- **Calls:** `open_wad_file_for_reading()`, `read_wad_header()`, `size_of_indexed_wad()`, `read_indexed_wad_from_file_into_buffer()`, `convert_wad_from_raw()`, etc.
- **Notes:** Blob format: 4-byte magic (0xDEADDEAD) + 4-byte length + packed header + raw WAD data; includes SIZEOF_encapsulated_wad_data overhead

---

## Control Flow Notes
This module is **not frame-bound**. It is a data-layer utility invoked at **load time** and **save time**:
- **Initialization:** Engine calls `open_wad_file_for_reading()` and `read_wad_header()` to validate/probe available WAD files
- **Level load:** `read_indexed_wad_from_file()` loads map/resource WADs; raw data is converted to `wad_data` (read-only)
- **Level save:** `create_empty_wad()` ΓåÆ `append_data_to_wad()` (multiple times) ΓåÆ `write_wad()` ΓåÆ `calculate_and_store_wadfile_checksum()`
- **Network:** `get_flat_data()` ΓåÆ send ΓåÆ `inflate_flat_data()` for level transfer in multiplayer
- **Shutdown:** `free_wad()` releases all allocated memory

## External Dependencies
- **Includes:**
  - `cseries.h` ΓÇö Platform abstraction (types, macros)
  - `tags.h` ΓÇö Tag type constants (POINT_TAG, MAP_INFO_TAG, etc.)
  - `crc.h` ΓÇö CRC checksum calculation
  - `game_errors.h` ΓÇö Error reporting (set_game_error)
  - `FileHandler.h` ΓÇö OpenedFile, FileSpecifier classes
  - `Packing.h` ΓÇö Binary packing/unpacking routines (StreamToValue, ValueToStream, etc.)
  - Standard: `<string.h>`, `<stdlib.h>`
- **External symbols (defined elsewhere):**
  - `level_transition_malloc()` ΓÇö Marathon-specific allocator for between-levels data
  - `set_game_error()`, `alert_out_of_memory()` ΓÇö Error handling
  - `calculate_crc_for_opened_file()` ΓÇö CRC32 computation
  - `dprintf()` ΓÇö Debugging output (dump_wad)
  - `obj_clear()`, `objlist_clear()`, `objlist_copy()` ΓÇö Generic struct/list utilities
  - OpenedFile methods: `SetPosition()`, `Read()`, `Write()`, `GetError()`, `Close()`
  - FileSpecifier methods: `Open()`, `Create()`, `GetName()`, `GetError()`
