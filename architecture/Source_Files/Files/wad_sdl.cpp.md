# Source_Files/Files/wad_sdl.cpp

## File Purpose
Implements SDL-based map file (WAD) discovery with support for both local filesystem and Steam Workshop sources. Provides two search strategies: by file checksum and by modification date.

## Core Responsibilities
- Locate WAD/map files by verifying checksums stored at file offset 0x44
- Find files matching specific modification dates
- Delegate to `FileFinder` base class for recursive directory traversal
- Integrate Steam Workshop item search with local data path search
- Convert game typecodes to Steam Workshop item types

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FindByChecksum` | class | Searches for files containing a specific 32-bit checksum at offset 0x44 |
| `FindByDate` | class | Searches for files matching a specific modification timestamp |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | external (shell_sdl.cpp) | Search path directories for local game files |
| `subscribed_workshop_items` | `vector<item_subscribed_query_result::item>` | external (Steam, conditional) | Subscribed Steam Workshop items available for search |

## Key Functions / Methods

### typecode_to_item_type
- **Signature:** `ItemType typecode_to_item_type(Typecode file_type)`
- **Purpose:** Map game file typecodes to Steam Workshop item classification
- **Inputs:** `file_type` ΓÇô game typecode (_typecode_scenario, _typecode_physics, etc.)
- **Outputs/Return:** Corresponding `ItemType` enum value (Map, Physics, Shapes, Sounds)
- **Side effects:** Asserts on unmapped typecodes
- **Calls:** (none visible)
- **Notes:** Only defined when `HAVE_STEAM` is set; handles specific typecodes only

### find_wad_file_that_has_checksum
- **Signature:** `bool find_wad_file_that_has_checksum(FileSpecifier &matching_file, Typecode file_type, short path_resource_id, uint32 checksum)`
- **Purpose:** Locate a map file containing a specific checksum value
- **Inputs:** Target `checksum`, file type, path resource ID (path_resource_id unused)
- **Outputs/Return:** Populates `matching_file`; returns true if found
- **Side effects:** Reads file headers from disk/Steam; creates temporary OpenedFile handles
- **Calls:** `typecode_to_item_type()`, `FindByChecksum::Find()` (inherited from FileFinder)
- **Notes:** Checksum read from big-endian uint32 at file offset 0x44; searches Steam items first, then local paths

### find_file_with_modification_date
- **Signature:** `bool find_file_with_modification_date(FileSpecifier &matching_file, Typecode file_type, short path_resource_id, TimeType modification_date)`
- **Purpose:** Locate a file matching a specific modification timestamp
- **Inputs:** Target `modification_date`, file type, path resource ID
- **Outputs/Return:** Populates `matching_file`; returns true if found
- **Side effects:** Queries file metadata; creates temporary FindByDate searcher
- **Calls:** `typecode_to_item_type()`, `FindByDate::Find()` (inherited from FileFinder)
- **Notes:** Search order mirrors checksum function: Steam Workshop first, then local data paths

## Control Flow Notes
These functions are likely called during game initialization or map loading to resolve asset references. Both delegate traversal to `FileFinder::Find()`, which recursively descends directories. Steam Workshop items are checked first (if `HAVE_STEAM`), then fallback to the ordered `data_search_path` vector.

## External Dependencies
- **FileHandler.h**: `FileSpecifier`, `OpenedFile`, `DirectorySpecifier` (file abstraction layer)
- **find_files.h**: `FileFinder` (base class for search implementations)
- **SDL2/SDL_endian.h**: `SDL_ReadBE32()` for endian-safe checksum reading
- **steamshim_child.h**: `ItemType` enum, `item_subscribed_query_result::item` struct (Steam integration, conditional)
- **cseries.h**: Standard includes, type definitions
