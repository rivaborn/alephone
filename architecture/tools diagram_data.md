# tools/dumprsrcmap.cpp
## File Purpose
Utility program to dump Macintosh resource maps from files. Reads a resource fork, enumerates all resource types and IDs, and prints their sizes to stdout. Designed for offline inspection of binary resource data.

## Core Responsibilities
- Parse command-line arguments and validate file input
- Open Macintosh resource files (raw, AppleSingle, or MacBinary formats)
- Parse the resource map to enumerate types and IDs
- Display resource metadata (type, ID, size) in human-readable format
- Provide string formatting utility matching game engine conventions

## External Dependencies
- **From included resource_manager.cpp**: `open_res_file()`, `get_resource_id_list()`, `get_resource()`, `count_resources()`, `cur_res_file_t`, `res_file_t::type_map_t`, `LoadedResource`
- **From included csalerts_sdl.cpp, Logging.cpp**: Logging macros and alert stubs (required for compilation but unused in this file)
- **SDL2**: `SDL_RWops` file handle abstraction
- **Standard C**: `<stdio.h>`, `<stdlib.h>`, `<stdarg.h>`
- **STL**: `<vector>`, nested `<map>` in resource_manager.cpp

# tools/dumpwad.cpp
## File Purpose

Standalone command-line utility for inspecting Marathon WAD (Where's All the Data) files. Reads the WAD file structure, parses the header and directory, and dumps all metadata and resource information in human-readable format. Used for debugging and analyzing level/asset files.

## Core Responsibilities

- Parse command-line arguments (WAD file path)
- Open and validate WAD files using the WAD I/O subsystem
- Read and display WAD header (version, checksum, resource count)
- Extract and display directory entries for each resource
- Parse and interpret level metadata (mission/environment flags, entry points, level names)
- Enumerate all tags (resource types) within each WAD entry
- Format and print all information to stdout with human-readable flag decoding

## External Dependencies

- **wad.cpp**: `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()`, `read_indexed_wad_from_file()`, `get_indexed_directory_data()`, `calculate_directory_offset()`, `free_wad()`
- **FileHandler_SDL.cpp** (via includes): File I/O abstraction (OpenedFile, FileSpecifier)
- **Packing.cpp**: `StreamToValue()`, `StreamToBytes()` (serialization macros)
- **crc.cpp**: CRC calculations (included but not directly used here)
- **Logging.cpp**: Logging support (included but stubbed out in tool)
- **map.h**: Data structure definitions (directory_data, etc.)
- Standard C: `<stdio.h>`, `<stdlib.h>`, `<stdarg.h>`

**Dummy declarations** (to avoid linker errors): `set_game_error()`, `dprintf()`, `alert_user()`, `level_transition_malloc()`


