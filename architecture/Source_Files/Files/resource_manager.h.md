# Source_Files/Files/resource_manager.h

## File Purpose

Header defining the resource management interface for loading game assets (sprites, sounds, maps, etc.) on non-Mac platforms. Uses SDL2 to abstract cross-platform file I/O while emulating Classic Mac resource file semantics.

## Core Responsibilities

- Initialize and manage a stack of open resource files
- Retrieve resources by type code + ID or type code + index
- Count and enumerate resources by type
- Support both current-file-only and cascading multi-file searches
- Handle external resource files for mod/extension support
- Provide SDL2-based I/O abstractions for resource data streams

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | class (forward decl) | File path/location specifier |
| LoadedResource | class (forward decl) | Container for loaded resource data |

## Global / File-Static State

None.

## Key Functions / Methods

### initialize_resources
- Signature: `void initialize_resources(void)`
- Purpose: Initialize the resource management subsystem
- Inputs: None
- Outputs/Return: void
- Side effects: Sets up global resource state, SDL structures
- Calls: Not inferable from this file
- Notes: Likely called once at engine startup

### open_res_file / open_res_file_from_rwops
- Signature: `SDL_RWops *open_res_file(FileSpecifier &file)` / `SDL_RWops *open_res_file_from_rwops(SDL_RWops *file)`
- Purpose: Open a resource file and push onto resource stack
- Inputs: File path (FileSpecifier) or existing SDL stream
- Outputs/Return: SDL_RWops handle (null on failure)
- Side effects: Opens file I/O, updates resource file stack
- Calls: Not inferable from this file

### close_res_file / cur_res_file / use_res_file
- Signature: `void close_res_file(SDL_RWops *)` / `SDL_RWops *cur_res_file(void)` / `void use_res_file(SDL_RWops *)`
- Purpose: Manage the active resource file context
- Inputs: SDL_RWops handle (for close/use)
- Outputs/Return: Current file handle (cur_res_file only)
- Side effects: Modify resource file stack state
- Calls: Not inferable from this file

### count_1_resources / count_resources
- Signature: `size_t count_1_resources(uint32 type)` / `size_t count_resources(uint32 type)`
- Purpose: Count resources of a given four-character type
- Inputs: Type code (uint32 four-char code)
- Outputs/Return: Resource count
- Side effects: None
- Notes: "1" variant searches current file only; non-"1" searches all open files

### get_*_resource / get_*_ind_resource
- Signature: `bool get_1_resource(uint32 type, int id, LoadedResource &rsrc)` / `bool get_resource(...)` / `bool get_1_ind_resource(uint32 type, int index, LoadedResource &rsrc)` / `bool get_ind_resource(...)`
- Purpose: Load resource by ID or by index; populate LoadedResource struct
- Inputs: Type code, ID or index, reference to output struct
- Outputs/Return: bool (success), LoadedResource via reference parameter
- Side effects: Allocates resource data
- Calls: Not inferable from this file
- Notes: By-ID vs by-index variants; "1" = current file, else cascading search

### get_1_resource_id_list / get_resource_id_list
- Signature: `void get_1_resource_id_list(uint32 type, vector<int> &ids)` / `void get_resource_id_list(...)`
- Purpose: Retrieve all IDs of a given resource type
- Inputs: Type code, reference to vector to populate
- Outputs/Return: void (output via reference)
- Side effects: Populates ids vector
- Calls: Not inferable from this file

### has_1_resource / has_resource
- Signature: `bool has_1_resource(uint32 type, int id)` / `bool has_resource(uint32 type, int id)`
- Purpose: Check if a resource exists
- Inputs: Type code, resource ID
- Outputs/Return: bool
- Side effects: None (read-only query)
- Calls: Not inferable from this file

### set_external_resources_file / close_external_resources
- Signature: `void set_external_resources_file(FileSpecifier &)` / `void close_external_resources(void)`
- Purpose: Manage supplemental external resource files
- Inputs: File specifier (set function)
- Outputs/Return: void
- Side effects: Manage external resource file state
- Calls: Not inferable from this file

## Control Flow Notes

Resources are initialized early (`initialize_resources`), then accessed throughout engine lifetime. The dual API ("1" vs. non-"1" functions) implements a resource file stack: current-file-only lookups for speed, cascading lookups for fallback/inheritance. External resources allow runtime extension without rebuilding the main resource archive.

## External Dependencies

- `cstypes.h` ΓÇö uint32, fixed-point types, platform macros
- `<SDL2/SDL.h>` ΓÇö SDL_RWops for cross-platform I/O
- `<vector>` ΓÇö std::vector for ID lists
- `<stdio.h>` ΓÇö Standard C I/O declarations
- **Forward declared**: FileSpecifier, LoadedResource (defined elsewhere)
