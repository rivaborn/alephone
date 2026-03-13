# Source_Files/Files/import_definitions.cpp

## File Purpose
Imports and unpacks physics definition data (monsters, effects, projectiles, weapons, physics constants) from external physics files. Supports both Marathon 1 legacy format and modern WAD-based physics files, with network transfer capabilities.

## Core Responsibilities
- Manage the active physics file specification and enable file selection
- Initialize all physics definition subsystems during game setup
- Detect physics file format (Marathon 1 vs. modern WAD format)
- Parse and unpack binary physics data from both file formats
- Prepare and process physics data for network transmission
- Calculate checksums for physics file validation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | external class | Represents a physics file location |
| `OpenedFile` | external class | File I/O wrapper for physics files |
| `wad_data` | external struct | In-memory unpacked WAD physics data |
| `wad_header` | external struct | WAD file header with metadata |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PhysicsFileSpec` | `FileSpecifier` | static | Currently active physics file location |

## Key Functions / Methods

### set_physics_file
- Signature: `void set_physics_file(FileSpecifier& File)`
- Purpose: Set the active physics file to load from
- Inputs: File specifier pointing to desired physics file
- Outputs/Return: None
- Side effects: Updates `PhysicsFileSpec` static variable
- Calls: None (direct assignment)

### set_to_default_physics_file
- Signature: `void set_to_default_physics_file(void)`
- Purpose: Reset physics file to the platform default
- Inputs: None
- Outputs/Return: None
- Side effects: Overwrites `PhysicsFileSpec` with default
- Calls: `get_default_physics_spec()`

### init_physics_wad_data
- Signature: `void init_physics_wad_data(void)`
- Purpose: Initialize all physics definition subsystems (monsters, effects, etc.)
- Inputs: None
- Outputs/Return: None
- Side effects: Calls init functions for monsters, effects, projectiles, weapons, physics constants
- Calls: `init_monster_definitions()`, `init_effect_definitions()`, `init_projectile_definitions()`, `init_physics_constants()`, `init_weapon_definitions()`

### physics_file_is_m1
- Signature: `bool physics_file_is_m1(void)`
- Purpose: Detect whether the active physics file uses Marathon 1 binary format
- Inputs: None (uses `PhysicsFileSpec`)
- Outputs/Return: `true` if file starts with M1 physics tag, `false` otherwise
- Side effects: Opens and closes the physics file
- Calls: `PhysicsFileSpec.Open()`, `SDL_ReadBE32()`, `PhysicsFile.Close()`
- Notes: Reads first 4 bytes (big-endian u32) and compares against M1 tag constants

### import_definition_structures
- Signature: `void import_definition_structures(void)`
- Purpose: Main entry point to load all physics data from active file
- Inputs: None (uses `PhysicsFileSpec`)
- Outputs/Return: None
- Side effects: Initializes physics subsystems; populates global physics data via unpacker functions
- Calls: `init_physics_wad_data()`, `physics_file_is_m1()`, `import_m1_physics_data()`, `get_physics_wad_data()`, `import_physics_wad_data()`, `free_wad()`

### get_network_physics_buffer
- Signature: `void *get_network_physics_buffer(int32 *physics_length)`
- Purpose: Serialize physics file into a buffer for network transmission
- Inputs: Pointer to length variable (output)
- Outputs/Return: Allocated buffer with physics data; output parameter set to buffer size in bytes
- Side effects: Allocates memory (caller must free); may modify error state
- Calls: `physics_file_is_m1()`, `PhysicsFileSpec.Open()`, `SDL_WriteBE32()`, `get_flat_data()`, `get_flat_data_length()`
- Notes: Prepends M1 magic cookie (0xDEAFDEAF) and length for M1 files; returns NULL on failure

### process_network_physics_model
- Signature: `void process_network_physics_model(void *data)`
- Purpose: Deserialize and apply physics data received from network
- Inputs: Serialized physics buffer (from `get_network_physics_buffer()`)
- Outputs/Return: None
- Side effects: Initializes physics subsystems; populates global physics data
- Calls: `init_physics_wad_data()`, `SDL_RWFromConstMem()`, `SDL_ReadBE32()`, `import_m1_physics_data_from_network()`, `inflate_flat_data()`, `import_physics_wad_data()`, `free_wad()`

### get_physics_file_checksum
- Signature: `uint32_t get_physics_file_checksum()`
- Purpose: Compute CRC32 checksum of active physics file
- Inputs: None (uses `PhysicsFileSpec`)
- Outputs/Return: 32-bit CRC value
- Side effects: None
- Calls: `calculate_crc_for_file()`

### get_physics_wad_data (static)
- Signature: `static struct wad_data *get_physics_wad_data(bool *bungie_physics)`
- Purpose: Internal function to open, read, and parse modern WAD-format physics file
- Inputs: Pointer to boolean (output: whether file is Bungie or Aleph One version)
- Outputs/Return: Allocated `wad_data` struct or NULL on failure
- Side effects: Resets game errors at end
- Calls: `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`, `set_game_error()`
- Notes: Checks for both `BUNGIE_PHYSICS_DATA_VERSION` and `PHYSICS_DATA_VERSION` headers

### import_physics_wad_data (static)
- Signature: `static void import_physics_wad_data(struct wad_data *wad)`
- Purpose: Extract and unpack five physics definition types from a parsed WAD
- Inputs: Parsed WAD data structure
- Outputs/Return: None
- Side effects: Calls unpacker functions that populate global physics arrays
- Calls: `extract_type_from_wad()` (5├ù), `unpack_monster_definition()`, `unpack_effect_definition()`, `unpack_projectile_definition()`, `unpack_physics_constants()`, `unpack_weapon_definition()`
- Notes: Asserts that extracted data length is a multiple of element size; checks count against limits

### import_m1_physics_data (static)
- Signature: `static void import_m1_physics_data()`
- Purpose: Parse Marathon 1 physics file format from disk
- Inputs: None (uses `PhysicsFileSpec`)
- Outputs/Return: None
- Side effects: Calls unpackers for M1 physics data
- Calls: `PhysicsFileSpec.Open()`, `PhysicsFile.GetLength()`, `PhysicsFile.Read()`, `AIStreamBE` constructor, `unpack_m1_monster_definition()`, etc.
- Notes: Loops through file reading 12-byte headers followed by data chunks; uses `AIStreamBE` for big-endian parsing

### import_m1_physics_data_from_network (static)
- Signature: `static void import_m1_physics_data_from_network(uint8 *data, uint32 length)`
- Purpose: Parse Marathon 1 physics format from a network buffer
- Inputs: Pointer to data buffer; buffer length in bytes
- Outputs/Return: None
- Side effects: Calls unpackers for M1 physics data
- Calls: `AIStreamBE` constructor, `unpack_m1_monster_definition()`, etc.
- Notes: Similar to `import_m1_physics_data()` but operates on in-memory buffer instead of file; loop index manually incremented

## Control Flow Notes
**Initialization**: `import_definition_structures()` is called during game startup to load physics definitions into memory.

**Network flow**: When joining a networked game, the host's physics file is serialized via `get_network_physics_buffer()`, transmitted, and clients deserialize via `process_network_physics_model()`.

**Format detection**: The code branches on `physics_file_is_m1()` to handle legacy (Marathon 1) vs. modern (WAD-based) physics formats. Both paths eventually call type-specific unpacker functions defined in included headers (monsters.h, weapons.h, etc.).

## External Dependencies
- **File I/O**: `FileHandler.h`, `wad.h`, `game_wad.h` (WAD file parsing)
- **Physics subsystems**: `monsters.h`, `effects.h`, `projectiles.h`, `weapons.h`, `physics_models.h` (init and unpacker functions)
- **Utilities**: `crc.h` (checksum), `tags.h` (data type constants), `AStream.h` (big-endian binary I/O)
- **Error handling**: `game_errors.h`
- **External symbols not defined here**: `get_default_physics_spec()`, all `init_*` and `unpack_*` functions, `extract_type_from_wad()`, `inflate_flat_data()`, `get_flat_data()`, etc.
- **SDL2**: `SDL_ReadBE32()`, `SDL_WriteBE32()`, `SDL_RWops` for endian-aware binary I/O
