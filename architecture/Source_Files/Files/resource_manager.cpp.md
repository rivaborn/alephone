# Source_Files/Files/resource_manager.cpp

## File Purpose
Implements MacOS resource fork handling for non-Mac platforms. Transparently reads resource data from AppleSingle, MacBinary II/III, and raw resource fork formats. Manages multiple open resource files with format auto-detection and a stack-like current-file context.

## Core Responsibilities
- Detect and parse AppleSingle and MacBinary container formats
- Parse MacOS resource map headers and reference lists from file offsets
- Maintain a list of open resource files with type-to-ID mappings
- Provide transparent resource lookup by type + ID or type + index
- Support multiple concurrent open files with a "current file" context
- Search across multiple open files in reverse stack order
- Handle initialization and cleanup via atexit hooks

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `res_file_t` | struct | Wraps an open resource file with typeΓåÆID map parsed from the resource header |
| `LoadedResource` | class (external) | Holds malloc'd resource data with size; defined in FileHandler.h |
| `type_map_t` | typedef | `map<uint32, id_map_t>` ΓÇö maps resource type (4-char code) to IDΓåÆoffset map |
| `id_map_t` | typedef | `map<int, uint32>` ΓÇö maps resource ID to file offset of resource data |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `res_file_list` | `list<res_file_t*>` | static (file) | List of all open resource files in order |
| `cur_res_file_t` | `list<res_file_t*>::iterator` | static (file) | Points to the current (top-of-stack) resource file for searches |
| `ExternalResources` | `OpenedResourceFile` | global | External terminal resources (Marathon 1 compatibility) |

## Key Functions / Methods

### is_applesingle
- Signature: `bool is_applesingle(SDL_RWops *f, bool rsrc_fork, int32 &offset, int32 &length)`
- Purpose: Detect and extract fork info from AppleSingle container format
- Inputs: File handle, fork type flag (true=resource, false=data), output references
- Outputs/Return: true if AppleSingle and fork found; sets offset and length
- Side effects: Seeks within file
- Notes: Checks magic `0x00051600` and version `0x00020000`; iterates fork entries until match

### is_macbinary
- Signature: `bool is_macbinary(SDL_RWops *f, int32 &data_length, int32 &rsrc_length)`
- Purpose: Detect MacBinary II/III format and extract fork lengths
- Inputs: File handle, output references for lengths
- Outputs/Return: true if valid MacBinary (CRC check passes); sets fork sizes
- Side effects: Reads 128-byte header and validates CRC-16
- Notes: Recognizes versions up to 0x81; validates header field ranges

### res_file_t::read_map
- Signature: `bool res_file_t::read_map(void)`
- Purpose: Parse the MacOS resource map (type list and reference lists) from an open file
- Inputs: None (operates on `this->f`)
- Outputs/Return: true on success; populates `types` map with typeΓåÆIDΓåÆoffset mappings
- Side effects: Seeks repeatedly; validates header integrity; logs via logTrace/logDump/logAnomaly
- Calls: `is_applesingle()`, `is_macbinary()`, SDL_RWseek, SDL_ReadBE32/16
- Notes: Handles format detection transparently; skips named resources; detects data/map offset corruption

### open_res_file_from_rwops
- Signature: `SDL_RWops* open_res_file_from_rwops(SDL_RWops* f)`
- Purpose: Wrap an open SDL_RWops, parse its resource map, and add to active file list
- Inputs: SDL file handle (or nullptr)
- Outputs/Return: The same file handle on success, nullptr on failure
- Side effects: Creates `res_file_t`, adds to `res_file_list`, sets as current; deletes object and closes file on parse failure
- Calls: `res_file_t::read_map()`
- Notes: Logs success/failure; updates `cur_res_file_t` iterator

### open_res_file
- Signature: `SDL_RWops* open_res_file(FileSpecifier &file)`
- Purpose: Main resource file opener; tries multiple file name variants and paths
- Inputs: FileSpecifier with base path
- Outputs/Return: SDL_RWops on success, nullptr if all attempts fail
- Side effects: Attempts sequential file opens (`.rsrc`, `.resources`, raw, `/..namedfork/rsrc`)
- Calls: `open_res_file_from_path()` repeatedly
- Notes: Graceful fallback chain for cross-platform resource location

### close_res_file
- Signature: `void close_res_file(SDL_RWops *file)`
- Purpose: Remove a resource file from the active list and clean up
- Inputs: SDL_RWops handle to close
- Outputs/Return: None
- Side effects: Erases from list, closes file, deletes res_file_t; updates cur_res_file_t if it was current
- Notes: Safe no-op if file not found; resets cur_res_file_t to end if list becomes empty

### get_resource (3 overloads)
- Signature: `bool res_file_t::get_resource(uint32 type, int id, LoadedResource &rsrc) const` (and two wrappers)
- Purpose: Load resource data by type and ID; allocates memory
- Inputs: Resource type (4-char code), ID, output reference
- Outputs/Return: true if found and loaded; rsrc.p and rsrc.size set; false otherwise
- Side effects: Calls `rsrc.Unload()` first; malloc's data; reads from file
- Calls: `type_map_t::find()`, `id_map_t::find()`, SDL_RWseek, SDL_ReadBE32
- Notes: Caller must free via `rsrc.Unload()` or destructor; multi-file wrapper searches current file backwards

### get_ind_resource (3 overloads)
- Signature: `bool res_file_t::get_ind_resource(uint32 type, int index, LoadedResource &rsrc) const` (and two wrappers)
- Purpose: Load resource by 1-based index within a type
- Inputs: Resource type, 1-based index, output reference
- Outputs/Return: true if index valid and loaded; rsrc populated
- Side effects: Allocates and reads like get_resource
- Notes: Index bounds checked; iterates id_map to reach index position

### count_resources (3 overloads)
- Signature: `size_t res_file_t::count_resources(uint32 type) const` (and two wrappers)
- Purpose: Count how many resources of a type exist
- Inputs: Resource type
- Outputs/Return: Count (0 if type not found)
- Notes: Multi-file wrapper counts current file backwards to beginning

### Helper Functions / Convenience Wrappers
- `get_resource_id_list()`, `get_1_resource_id_list()` ΓÇö populate vector with IDs of a type
- `has_resource()`, `has_1_resource()` ΓÇö boolean check for resource existence
- `cur_res_file()` ΓÇö get current SDL_RWops handle
- `use_res_file()` ΓÇö set current file by handle
- `find_res_file_t()` ΓÇö internal lookup of res_file_t by SDL_RWops

## Control Flow Notes

**Initialization:** `initialize_resources()` registers `close_external_resources()` as atexit handler for cleanup.

**Open ΓåÆ Read Map ΓåÆ Add to Stack:** `open_res_file()` attempts multiple filename variants; on first successful open and map parse, the file is pushed onto `res_file_list` and becomes current.

**Resource Lookup:** Default operations (no `_1_` suffix) search from current file backwards to the beginning of the list. This mimics MacOS resource fork stacking where later-opened files shadow earlier ones.

**Cleanup:** `close_res_file()` pops from the stack; if no files remain, `cur_res_file_t` is reset.

**External Resources:** `ExternalResources` (OpenedResourceFile wrapper) holds Terminal definitions and is managed separately via `set_external_resources_file()`.

## External Dependencies

- **SDL2** (`SDL_endian.h`, SDL_RWops functions): Cross-platform file I/O and byte order utilities
- **cseries.h**: Aleph One platform/type definitions (uint32, int32, etc.)
- **FileHandler.h**: FileSpecifier, LoadedResource, OpenedResourceFile classes
- **Logging.h**: logTrace, logNote, logAnomaly, logDump macros
- **Standard C++**: `<vector>`, `<list>`, `<map>`, `<stdio.h>`, `<string>`
