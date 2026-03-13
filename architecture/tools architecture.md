# Subsystem Overview

## Purpose
The tools subsystem provides command-line utilities for offline inspection and debugging of Marathon/Aleph One data files and binary resources. These standalone programs analyze WAD files and Macintosh resource maps, extracting and displaying file structure metadata in human-readable format for analysis and troubleshooting.

## Key Files
| File | Role |
|------|------|
| dumprsrcmap.cpp | Utility to parse and display Macintosh resource map metadata (types, IDs, sizes) |
| dumpwad.cpp | Utility to parse and display Marathon WAD file structure, headers, directories, and level metadata |

## Core Responsibilities
- Parse command-line arguments and validate file input paths
- Open and read Marathon data files (resource forks in raw/AppleSingle/MacBinary formats; WAD files)
- Parse binary file structures (resource maps, WAD headers, directory entries, level metadata)
- Extract and interpret resource/level metadata (types, IDs, sizes, mission/environment flags, entry points)
- Enumerate data elements (resource types and IDs; WAD tags and entries)
- Decode and display human-readable flag interpretations
- Output inspection results to stdout in formatted text

## Key Interfaces & Data Flow
**Consumes from:**
- resource_manager.cpp: `open_res_file()`, `get_resource_id_list()`, `get_resource()`, `count_resources()`, resource map parsing
- wad.cpp: `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()`, WAD I/O functions
- FileHandler_SDL.cpp: File I/O abstraction (OpenedFile, FileSpecifier)
- Packing.cpp: `StreamToValue()`, `StreamToBytes()` serialization utilities
- SDL2: `SDL_RWops` file handle abstraction
- map.h: Data structure definitions (directory_data, etc.)

**Exposes to:**
- Diagnostic output to stdout (human-readable file inspection data)

## Runtime Role
These are standalone offline utilities, not integrated into game runtime. Executed independently via command line; do not participate in game initialization, frame loops, or shutdown cycles.

## Notable Implementation Details
- Both utilities include dummy symbol declarations (set_game_error, dprintf, alert_user, level_transition_malloc) to avoid linker errors while remaining decoupled from main game runtime
- dumprsrcmap.cpp includes logging macros and alert stubs required for compilation but unused in the utility
- Support for multiple resource file formats (raw, AppleSingle, MacBinary) and Marathon WAD format
