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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `wad_header` | struct | WAD file header with version, checksums, counts, directory offset |
| `directory_entry` / `old_directory_entry` | struct | Maps resource index to file offset and length |
| `directory_data` | struct | Level metadata: mission/environment/entry-point flags, level name |
| `wad_data` | struct | Loaded WAD resource with tag array |
| `tag_data` | struct | Individual resource tag with type, length, data pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `local_data_dir`, `preferences_dir`, `saved_games_dir`, `recordings_dir` | DirectorySpecifier | global | Standard game directories (stubbed for tool) |
| `data_search_path` | vector\<DirectorySpecifier\> | global | Search path for resource files |
| `temporary` | char[256] | global | Temporary string buffer |

## Key Functions / Methods

### main
- Signature: `int main(int argc, char **argv)`
- Purpose: Entry point; orchestrates WAD file inspection and output
- Inputs: Command-line arguments (WAD file path required)
- Outputs/Return: 0 on success, 1 on error
- Side effects: Opens file, reads data, prints to stdout
- Calls: `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()`, `calculate_directory_offset()`, `read_indexed_wad_from_file()`, `get_indexed_directory_data()`, `unpack_directory_data()`
- Notes: Loops through wad_count entries; handles both old and new directory entry formats

### unpack_directory_data
- Signature: `static void unpack_directory_data(uint8 *Stream, directory_data *Objects)`
- Purpose: Deserialize directory data from byte stream (from WAD file)
- Inputs: Byte stream pointer, target directory_data struct
- Outputs/Return: None; modifies Objects in-place
- Side effects: Advances stream pointer
- Calls: `StreamToValue()`, `StreamToBytes()` (packing macros)
- Notes: Extracts mission flags, environment flags, entry-point flags, level name

### csprintf
- Signature: `char *csprintf(char *buffer, const char *format, ...)`
- Purpose: Formatted string printing to buffer (like sprintf)
- Inputs: Buffer, format string, variadic args
- Outputs/Return: Pointer to buffer
- Side effects: Writes to buffer
- Calls: `vsprintf()`
- Notes: Wrapper around vsprintf for convenience

## Control Flow Notes

Strictly linear utility flow (not part of game loop): command-line validation ΓåÆ file open ΓåÆ header parse ΓåÆ loop over directory entries (parse metadata, read tags) ΓåÆ print results. No dynamic state or frame-based updates.

## External Dependencies

- **wad.cpp**: `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()`, `read_indexed_wad_from_file()`, `get_indexed_directory_data()`, `calculate_directory_offset()`, `free_wad()`
- **FileHandler_SDL.cpp** (via includes): File I/O abstraction (OpenedFile, FileSpecifier)
- **Packing.cpp**: `StreamToValue()`, `StreamToBytes()` (serialization macros)
- **crc.cpp**: CRC calculations (included but not directly used here)
- **Logging.cpp**: Logging support (included but stubbed out in tool)
- **map.h**: Data structure definitions (directory_data, etc.)
- Standard C: `<stdio.h>`, `<stdlib.h>`, `<stdarg.h>`

**Dummy declarations** (to avoid linker errors): `set_game_error()`, `dprintf()`, `alert_user()`, `level_transition_malloc()`
