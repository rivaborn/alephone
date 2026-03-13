# Source_Files/Files/game_wad.h

## File Purpose
Declares functions for managing game WAD (world data) filesΓÇösaving/loading game state, handling map checksums, and orchestrating level and player data persistence. Serves as the interface between file I/O and the game world's dynamic state structures.

## Core Responsibilities
- Save and load game files with metadata/image data
- Build and validate WAD structures for map/game state
- Map file management and checksum matching
- Extract dynamic game state and player data from saved WAD
- Level export and default save game configuration
- Level metadata queries (physics, Lua script presence)

## Key Types / Data Structures
None (this is a pure declaration header).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SAVE_GAME_METADATA_INDEX` | `const int` | static | Index constant for metadata chunk in WAD |

## Key Functions / Methods

### save_game_file
- Signature: `bool save_game_file(FileSpecifier& File, const std::string& metadata, const std::string& imagedata)`
- Purpose: Persist current game state to disk file with metadata and screenshot
- Inputs: File destination, metadata string, image data string
- Outputs/Return: `true` if successful, `false` otherwise
- Side effects: File I/O; creates/overwrites game save file

### build_meta_game_wad
- Signature: `struct wad_data *build_meta_game_wad(const std::string& metadata, const std::string& imagedata, struct wad_header *header, int32 *length)`
- Purpose: Construct WAD structure containing game metadata and image data
- Inputs: Metadata and image strings; output pointers for WAD header and length
- Outputs/Return: Pointer to constructed `wad_data`; populates header and length
- Side effects: Memory allocation for WAD structure

### process_map_wad
- Signature: `bool process_map_wad(struct wad_data *wad, bool restoring_game, short version)`
- Purpose: Deserialize map and game state from WAD; used when loading levels or resuming games
- Inputs: WAD data; flag indicating restore vs. fresh load; version number
- Outputs/Return: `true` if successful
- Side effects: Populates global world structures; initializes level geometry and objects
- Notes: Exposed for netgame-resuming code

### match_checksum_with_map
- Signature: `bool match_checksum_with_map(short vRefNum, long dirID, uint32 checksum, FileSpecifier& File)`
- Purpose: Locate map file on disk whose checksum matches a given value; validates scenario integrity
- Inputs: Filesystem reference, directory ID, checksum to match
- Outputs/Return: `true` if found; populates `File`
- Side effects: File system search

### set_map_file
- Signature: `void set_map_file(FileSpecifier& File, bool runScript = true)`
- Purpose: Set current active map file; optionally execute initialization scripts
- Inputs: Map file; script execution flag (default `true`)
- Side effects: Updates global map file reference; may execute map scripts

### get_dynamic_data_from_save
- Signature: `dynamic_data get_dynamic_data_from_save(FileSpecifier& File)`
- Purpose: Extract dynamic game state (ticks, random seed, object counts, etc.) from save file
- Inputs: Save file to read
- Outputs/Return: Populated `dynamic_data` structure
- Side effects: File I/O

### get_dynamic_data_from_wad
- Signature: `bool get_dynamic_data_from_wad(wad_data* wad, dynamic_data* dest)`
- Purpose: Deserialize dynamic game state from WAD structure
- Inputs: WAD containing state data
- Outputs/Return: `true` if successful; populates `dest`

### get_player_data_from_wad
- Signature: `bool get_player_data_from_wad(wad_data* wad)`
- Purpose: Extract and restore player state from WAD
- Inputs: WAD data
- Outputs/Return: `true` if successful
- Side effects: Updates global player arrays/structures

### get_map_file
- Signature: `FileSpecifier& get_map_file(void)`
- Purpose: Retrieve current active map file
- Outputs/Return: Reference to current map file
- Notes: Exposed for external map-switching code

### level_has_embedded_physics_lua
- Signature: `void level_has_embedded_physics_lua(int Level, bool& HasPhysics, bool& HasLua)`
- Purpose: Query whether a level has embedded physics model or Lua scripts
- Inputs: Level index
- Outputs/Return: Reference outputs for physics and Lua flags

## Control Flow Notes
Part of the load/save pipeline during:
- Game initialization: `new_game()` ΓåÆ load level via `set_map_file()` and `process_map_wad()`
- Saving: `save_game_file()` ΓåÆ builds WAD via `build_meta_game_wad()`
- Resuming/netgame restore: `process_map_wad()` ΓåÆ restore `dynamic_data` and player state
- Level transitions: Map checksums validated via `match_checksum_with_map()`; state extracted via `get_dynamic_data_from_wad()`

## External Dependencies
- **Includes**: `cstypes.h` (integer types), `map.h` (world structures), `<string>` (std::string)
- **Forward declarations**: `FileSpecifier`, `wad_data`, `wad_header` (defined elsewhere)
- **Symbols used but not defined**: `dynamic_data`, `wad_data` (from map.h and WAD module)
