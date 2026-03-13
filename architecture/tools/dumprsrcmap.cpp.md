# tools/dumprsrcmap.cpp

## File Purpose
Utility program to dump Macintosh resource maps from files. Reads a resource fork, enumerates all resource types and IDs, and prints their sizes to stdout. Designed for offline inspection of binary resource data.

## Core Responsibilities
- Parse command-line arguments and validate file input
- Open Macintosh resource files (raw, AppleSingle, or MacBinary formats)
- Parse the resource map to enumerate types and IDs
- Display resource metadata (type, ID, size) in human-readable format
- Provide string formatting utility matching game engine conventions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `res_file_t` | struct | Encapsulates opened resource file, type/ID maps, and file I/O state (from resource_manager.cpp) |
| `type_map_t` | typedef | `map<uint32, id_map_t>` ΓÇö maps resource type codes to their ID maps |
| `id_map_t` | typedef | `map<int, uint32>` ΓÇö maps resource ID to offset in file |
| `LoadedResource` | struct | Holds resource data pointer and size (defined elsewhere) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `local_data_dir` | DirectorySpecifier | file-static | Stub; required by linked resource_manager.cpp |
| `preferences_dir` | DirectorySpecifier | file-static | Stub; required by linked resource_manager.cpp |
| `saved_games_dir` | DirectorySpecifier | file-static | Stub; required by linked resource_manager.cpp |
| `recordings_dir` | DirectorySpecifier | file-static | Stub; required by linked resource_manager.cpp |
| `data_search_path` | `vector<DirectorySpecifier>` | file-static | Stub; required by linked resource_manager.cpp |

## Key Functions / Methods

### main
- **Signature**: `int main(int argc, char **argv)`
- **Purpose**: Dump all resources from a Macintosh resource fork file
- **Inputs**: `argc` (must be ΓëÑ 2), `argv[1]` (file path as C string)
- **Outputs/Return**: 0 on success; exits with code 1 on file open or argument failure
- **Side effects**: Reads file, allocates/deallocates LoadedResource objects, writes to stdout
- **Calls**: `open_res_file()`, `get_resource_id_list()`, `get_resource()`, `count_resources()`
- **Notes**: Assumes `cur_res_file_t` is valid after `open_res_file()` succeeds; prints resource type as ASCII fourcc + hex

### csprintf
- **Signature**: `char *csprintf(char *buffer, const char *format, ...)`
- **Purpose**: sprintf wrapper matching game engine conventions (likely used by included modules)
- **Inputs**: pre-allocated `buffer`, printf-style format string and varargs
- **Outputs/Return**: pointer to `buffer`
- **Side effects**: Writes formatted string to buffer
- **Calls**: `va_start()`, `vsprintf()`, `va_end()`

### set_game_error
- **Signature**: `void set_game_error(short a, short b)`
- **Purpose**: Dummy stub to satisfy linker; game engine sets error state elsewhere
- **Inputs**: Two short integers (semantics unknown)
- **Outputs/Return**: void; no-op
- **Notes**: Present only to avoid undefined symbol errors when linking resource_manager.cpp

## Control Flow Notes
1. **Startup**: `main()` validates arguments, opens file via resource manager
2. **Enumeration**: Walks `res_file_t::types` map to get all resource types
3. **Per-Type**: Calls `count_resources()` for metadata, `get_resource_id_list()` to list IDs
4. **Per-ID**: Calls `get_resource()` to fetch size, immediately `Unload()` (memory freed)
5. **Shutdown**: Program exits naturally; file closed by underlying SDL/resource manager cleanup

This is a one-shot utility with no frame loop, update/render, or persistent state.

## External Dependencies
- **From included resource_manager.cpp**: `open_res_file()`, `get_resource_id_list()`, `get_resource()`, `count_resources()`, `cur_res_file_t`, `res_file_t::type_map_t`, `LoadedResource`
- **From included csalerts_sdl.cpp, Logging.cpp**: Logging macros and alert stubs (required for compilation but unused in this file)
- **SDL2**: `SDL_RWops` file handle abstraction
- **Standard C**: `<stdio.h>`, `<stdlib.h>`, `<stdarg.h>`
- **STL**: `<vector>`, nested `<map>` in resource_manager.cpp
