# Source_Files/Files/AStream.cpp
## File Purpose
Implements serialization/deserialization stream classes for network communication and data persistence. Provides explicit big-endian and little-endian byte ordering support with bounds checking and exception handling.

## Core Responsibilities
- Implement extraction operators (`>>`) for input stream deserialization across signed/unsigned types
- Implement insertion operators (`<<`) for output stream serialization
- Provide separate big-endian and little-endian implementations for multi-byte types
- Perform bounds checking on stream positions with optional exception throwing
- Define exception class for serialization failures with proper resource cleanup

## External Dependencies
- `<string.h>`: `memcpy()`, `strdup()`, `free()`
- `<string>`: `std::string` (used in exception constructor)
- `<exception>`: `std::exception` (base class for `failure`)
- `"AStream.h"`: Class declarations and `basic_astream<T>` template
- `"cstypes.h"`: Type aliases (`uint8`, `int8`, `uint16`, `int16`, `uint32`, `int32`)
- Conditional compilation guard: `#if !defined(DISABLE_NETWORKING)`

# Source_Files/Files/AStream.h
## File Purpose
Provides typed, endian-aware binary serialization/deserialization streams for game data. Replaces the less-clear `Packing.h` from AlephOne with explicit type safety and compile-time endian selection via subclassing rather than preprocessor directives.

## Core Responsibilities
- Define templated base stream class with state/bounds tracking and exception handling
- Provide input stream classes (`AIStream*`) for binary deserialization with operator>>
- Provide output stream classes (`AOStream*`) for binary serialization with operator<<
- Support both Big Endian (BE) and Little Endian (LE) byte order conversion
- Track stream position, read bounds, error states (good/bad/fail), and exception masks
- Enable safe array/buffer reading and writing via template `read()` / `write()` methods

## External Dependencies
- `<string>`, `<exception>` ΓÇô Standard C++ for `failure` exception class
- `"cstypes.h"` ΓÇô Custom fixed-width integer typedefs (`uint8`, `int16`, `uint32`, etc.)

# Source_Files/Files/crc.cpp
## File Purpose
Provides CRC (Cyclic Redundancy Check) computation utilities for both files and memory buffers. Implements two algorithms: CRC32 (using polynomial-based lookup table) and CCITT CRC16 (using pre-computed table). Used for integrity verification and checksumming in the game engine.

## Core Responsibilities
- Compute CRC32 checksums for files via FileSpecifier abstraction
- Compute CRC32 checksums for in-memory buffers incrementally
- Compute CCITT CRC16 checksums for in-memory buffers
- Build and manage dynamic lookup table for efficient CRC32 calculation
- Handle chunked file I/O with preserved file position
- Provide both high-level file-based and low-level buffer-based APIs

## External Dependencies
- **Notable includes:** `cseries.h` (type definitions), `FileHandler.h` (FileSpecifier, OpenedFile abstractions), `crc.h` (public interface)
- **Defined elsewhere:** `FileSpecifier` class, `OpenedFile` class, `uint32` / `uint16` types, `byte` typedef, `assert` macro
- **Standard library:** `stdlib.h` (memory allocation)

# Source_Files/Files/crc.h
## File Purpose
Header declaring CRC (Cyclic Redundancy Check) calculation functions for the Aleph One game engine. Provides 32-bit and 16-bit CRC computation over files and raw data buffers for integrity verification.

## Core Responsibilities
- Declare CRC-32 calculation for FileSpecifier-based files
- Declare CRC-32 calculation for OpenedFile objects
- Declare CRC-32 calculation for raw byte buffers
- Declare CRC-16 (CCITT variant) calculation for raw byte buffers
- Provide data integrity checking utilities

## External Dependencies
- `cstypes.h` ΓÇö Standard type definitions (uint32, uint16, int32, unsigned char)
- `FileSpecifier` class ΓÇö defined elsewhere; game's file abstraction layer
- `OpenedFile` class ΓÇö defined elsewhere; game's file handle abstraction

# Source_Files/Files/extensions.h
## File Purpose
Header interface for physics file management in the Aleph One game engine (Marathon engine port). Provides functions to load physics definitions from files, handle network physics data, and compute file checksums for validation.

## Core Responsibilities
- Set and configure physics data files for the engine
- Import and process physics definition structures from files
- Manage network-synchronized physics buffers
- Validate physics data integrity via checksums
- Support version control of physics data formats

## External Dependencies
- **cstypes.h** ΓÇö provides `int32`, `uint32_t` fixed-size integer types via SDL_types.h
- **FileSpecifier** ΓÇö forward-declared class; implementation defined elsewhere (likely in file manager / OS abstraction layer)

---

**Version Constants:**
- `BUNGIE_PHYSICS_DATA_VERSION = 0` (legacy)
- `PHYSICS_DATA_VERSION = 1` (current)

# Source_Files/Files/FileHandler.cpp
## File Purpose
Cross-platform file I/O abstraction and dialog system for the Aleph One game engine. Wraps SDL_RWops and boost.filesystem to provide path manipulation, file operations, resource fork handling, and native/custom file selection dialogs with ZIP archive support.

## Core Responsibilities
- Manage opened files, resources, and resource forks via SDL_RWops abstractions
- Transparently handle macOS compatibility formats (AppleSingle, MacBinary) on read
- Detect file types by extension or content inspection (magic numbers)
- Provide FileSpecifier class for path construction, validation, and file operations
- Support ZIP archive enumeration (ZZIP integration)
- Implement file selection dialogs (native NFD or custom SDL-based fallback)
- Maintain per-user directory specifications (data, preferences, saves, cache, recordings)
- Provide directory browsing widget with sorting and navigation

## External Dependencies

**Notable includes**:
- SDL2 (SDL_RWops, SDL_RWseek, SDL endian macros)
- boost::filesystem (directory_iterator, path, is_directory, last_write_time, remove, rename, create_directory)
- boost::iostreams (seekable_device_tag, stream_offset)
- boost::algorithm (ends_with)
- ZZIP (zzip_dir_open_ext_io, zzip_dir_read; optional)
- NFD (native file dialogs; optional)

**External symbols (defined elsewhere)**:
- Global directories: data_search_path, local_data_dir, preferences_dir, saved_games_dir, quick_saves_dir, image_cache_dir, recordings_dir (from shell_sdl.cpp)
- Mac format detection: is_applesingle, is_macbinary
- UI framework: dialog, w_list, w_button, w_select, w_title, w_static_text, w_text_entry, vertical_placer, horizontal_placer (sdl_dialogs.h, sdl_widgets.h)
- Drawing: draw_text, get_theme_color, font, set_drawing_clip_rectangle (sdl_widgets.h)
- Screen: get_screen_mode, toggle_fullscreen, MainScreenWindow, update_game_window (screen.h, interface.h)
- Preferences: environment_preferences (global settings object)
- Audio: play_dialog_sound (SoundManager.h)

# Source_Files/Files/FileHandler.h
## File Purpose
Provides platform-independent file, directory, and resource I/O abstractions for the Aleph One game engine. Uses SDL_RWops for underlying I/O and abstracts macOS resource fork patterns across platforms.

## Core Responsibilities
- Abstraction of file I/O operations (open, read, write, seek, close, position/length queries)
- Management of loaded resources with automatic deallocation on destruction
- File specification and path resolution (including data-directory search)
- Directory enumeration and ZIP archive listing
- Resource fork abstraction (primarily for macOS compatibility)
- File metadata queries (existence, type, modification date)
- File operations (copy, delete, rename, create directories)
- Platform-specific file dialogs (both synchronous and asynchronous)

## External Dependencies

**Includes:**
- `tags.h` ΓÇô Typecode enum and file-type tag constants
- `<SDL2/SDL.h>` ΓÇô SDL_RWops for platform-independent I/O
- `<boost/iostreams/categories.hpp>`, `<boost/iostreams/positioning.hpp>` ΓÇô Boost.IOStreams integration
- `<vector>`, `<string>` ΓÇô STL containers
- `<time.h>` ΓÇô TimeType and time_t
- `<errno.h>` ΓÇô Error codes
- `<stddef.h>` ΓÇô size_t

**External symbols (defined elsewhere):**
- `SDL_RWops` ΓÇô SDL file abstraction struct
- `Typecode` enum, `get_typecode()`, `set_typecode()`, `FOUR_CHARS_TO_INT()` ΓÇô From tags.h
- `TimeType` ΓÇô Type alias, likely from cstypes.h
- Standard library I/O and allocation

# Source_Files/Files/find_files.h
## File Purpose
Defines an abstract file-finder framework and a concrete implementation for collecting files from the filesystem. Uses a template-method pattern where `FileFinder` conducts recursive directory traversal and invokes a virtual callback for each file found.

## Core Responsibilities
- Abstract base class `FileFinder` that defines the contract for file search operations with recursive directory support
- Virtual callback `found()` that subclasses override to handle discovered files
- Concrete implementation `FindAllFiles` that appends all discovered files to a provided `std::vector`
- Support for wildcard file type matching via `WILDCARD_TYPE` constant

## External Dependencies
- **FileHandler.h** ΓÇô provides `FileSpecifier`, `DirectorySpecifier`, `Typecode` constants, and file I/O abstractions
- **tags.h** (via FileHandler.h) ΓÇô defines typecode constants (e.g., `_typecode_unknown`)
- **\<vector\>** ΓÇô standard library container for file results

# Source_Files/Files/find_files_sdl.cpp
## File Purpose
Implements recursive file discovery for the Aleph One game engine's SDL backend. Provides breadth-first directory traversal with typecode-based file filtering and callback-driven result collection.

## Core Responsibilities
- Implements breadth-first recursive directory traversal using a queue
- Filters files by typecode (or returns all files if `WILDCARD_TYPE` specified)
- Skips engine-specific directories ("Plugins", "Scenarios") at root level
- Invokes virtual callback method for each matching file found
- Implements concrete callback for `FindAllFiles` to collect results into a vector

## External Dependencies
- **FileHandler.h:** `FileSpecifier`, `DirectorySpecifier` (typedef alias), `dir_entry`
- **find_files.h:** `FileFinder` base class, `FindAllFiles` derived class, `WILDCARD_TYPE` constant
- **cseries.h:** Portability macros and includes
- **STL:** `<vector>`, `<algorithm>`, `<queue>`

# Source_Files/Files/game_wad.cpp
## File Purpose
Manages loading, saving, and persistence of game maps and save states from WAD (archive) files. Orchestrates level initialization, game setup, network map distribution, and save/restore functionality for the game engine.

## Core Responsibilities
- Map file management (set, load, validate by checksum)
- Level loading from WAD files with geometry and entity placement
- New game initialization with player setup and level entry
- Save game creation and restoration (full game state serialization)
- Level entry point querying (spawn locations, mission types)
- Network map synchronization and distribution
- Dynamic memory allocation for map structures based on loaded counts
- Packing/unpacking of game state for persistence

## External Dependencies
- **Map/World:** map.h, world.h ΓÇö defines all geometry and entity structures
- **Game Systems:** monsters.h, projectiles.h, effects.h, player.h, platforms.h ΓÇö entity management
- **Networking:** network.h ΓÇö map sync, player data distribution
- **I/O:** FileHandler.h, wad.h, Packing.h ΓÇö file and data serialization
- **Scripting:** XML_LevelScript.h ΓÇö MML level script loading
- **UI:** interface.h, game_window.h, shell.h ΓÇö error reporting, shell interaction
- **Other:** lightsource.h, media.h, weapons.h, scenery.h, computer_interface.h, images.h, SoundManager.h, Plugins.h, ChaseCam.h, render.h, motion_sensor.h, Music.h, ephemera.h
- **Defined Elsewhere:** `process_map_wad()`, `entering_map()`, `leaving_map()`, `setup_revert_game_info()`, `recreate_players_for_new_level()`, `get_flat_data()`, `inflate_flat_data()` etc.

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

## External Dependencies
- **Includes**: `cstypes.h` (integer types), `map.h` (world structures), `<string>` (std::string)
- **Forward declarations**: `FileSpecifier`, `wad_data`, `wad_header` (defined elsewhere)
- **Symbols used but not defined**: `dynamic_data`, `wad_data` (from map.h and WAD module)

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

## External Dependencies
- **File I/O**: `FileHandler.h`, `wad.h`, `game_wad.h` (WAD file parsing)
- **Physics subsystems**: `monsters.h`, `effects.h`, `projectiles.h`, `weapons.h`, `physics_models.h` (init and unpacker functions)
- **Utilities**: `crc.h` (checksum), `tags.h` (data type constants), `AStream.h` (big-endian binary I/O)
- **Error handling**: `game_errors.h`
- **External symbols not defined here**: `get_default_physics_spec()`, all `init_*` and `unpack_*` functions, `extract_type_from_wad()`, `inflate_flat_data()`, `get_flat_data()`, etc.
- **SDL2**: `SDL_ReadBE32()`, `SDL_WriteBE32()`, `SDL_RWops` for endian-aware binary I/O

# Source_Files/Files/Packing.cpp
## File Purpose

Implements binary serialization primitives for converting 16-bit and 32-bit integers to/from byte streams. Supports both big-endian (BE) and little-endian (LE) byte orders with sign-aware handling for signed and unsigned variants. Used by Marathon's game engine to pack/unpack network and file data.

## Core Responsibilities

- Convert big-endian byte streams to 16-bit/32-bit unsigned integers
- Convert big-endian byte streams to 16-bit/32-bit signed integers
- Convert 16-bit/32-bit integers to big-endian byte streams
- Convert little-endian byte streams to 16-bit/32-bit unsigned integers
- Convert little-endian byte streams to 16-bit/32-bit signed integers
- Convert 16-bit/32-bit integers to little-endian byte streams
- Advance stream pointers by consumed/written byte count

## External Dependencies

- **cseries.h** ΓåÆ includes cstypes.h (defines uint8, uint16, int16, uint32, int32)
- **Packing.h** ΓåÆ header declaring extern function prototypes and macro dispatch logic
- **DEFINE_PACKING_INTERNAL** ΓåÆ signals to Packing.h not to inline certain utilities (implementation moved to .cpp to resolve inlining issues per Aug 2002 note)

# Source_Files/Files/Packing.h
## File Purpose
Provides serialization utilities to convert between in-memory data structures (with native alignment) and packed big-endian byte streams, as used by the Marathon series. Handles endianness conversion and stream pointer advancement during pack/unpack operations.

## Core Responsibilities
- Unpack numerical values (16/32-bit int/uint) from byte streams via `StreamToValue()`
- Pack numerical values into byte streams via `ValueToStream()`
- Serialize/deserialize arrays of values via `StreamToList()` and `ListToStream()`
- Handle raw byte block packing/unpacking without interpretation
- Support configurable endianness (big-endian default, overridable to little-endian)
- Maintain stream pointer advancement across all operations

## External Dependencies
- `cstypes.h`: Provides sized integer typedefs (uint8, int16, uint16, int32, uint32)
- `memcpy`: Standard C library for raw byte copying
- `Packing.cpp`: Contains implementations of `StreamToValue` and `ValueToStream` overloads


# Source_Files/Files/preprocess_map_sdl.cpp
## File Purpose
SDL implementation for save game routines and default resource file discovery. Handles locating essential game data files (maps, shapes, sounds) in the search path, managing game save/load operations, and verifying default resources exist at startup.

## Core Responsibilities
- Locate default game data files (map, physics, shapes, sounds, music, theme) via data search path
- Verify essential default resources exist (`have_default_files()`)
- Manage game save operations with user feedback
- Present save game selection dialogs for loading
- Support post-processing hooks for save file metadata (currently stubbed for macOS compatibility)

## External Dependencies
- **Includes:** `cseries.h` (core macros, types), `FileHandler.h` (FileSpecifier), `world.h` (coordinate types), `map.h` (map data), `shell.h` (application shell), `interface.h` (resource strings, UI), `game_wad.h` (save format), `game_errors.h` (error system), `QuickSave.h` (save dialogs/operations)
- **External symbols:** `data_search_path` (from shell_sdl.cpp), `getcstr()`, `alert_bad_extra_file()`, `load_quick_save_dialog()`, `create_quick_save()`, `screen_printf()` (defined elsewhere)
- **Standard library:** `<vector>`, `<string>`

# Source_Files/Files/preprocess_map_shared.cpp
## File Purpose
Provides a simplified auto-save wrapper for single-player games. The function delegates to the standard save mechanism, eliminating the need for dialog presentation and using standardized overwrite logic.

## Core Responsibilities
- Provide an entry point for auto-saving in single-player mode
- Delegate auto-save operations to the core `save_game()` function
- Maintain backward compatibility with callers expecting `save_game_full_auto()`

## External Dependencies
- **Includes:** `interface.h` ΓÇô provides the declaration and (presumed) implementation of `save_game()`
- **Symbols used but defined elsewhere:** `save_game()` function

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

## External Dependencies

- **SDL2** (`SDL_endian.h`, SDL_RWops functions): Cross-platform file I/O and byte order utilities
- **cseries.h**: Aleph One platform/type definitions (uint32, int32, etc.)
- **FileHandler.h**: FileSpecifier, LoadedResource, OpenedResourceFile classes
- **Logging.h**: logTrace, logNote, logAnomaly, logDump macros
- **Standard C++**: `<vector>`, `<list>`, `<map>`, `<stdio.h>`, `<string>`

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

## External Dependencies

- `cstypes.h` ΓÇö uint32, fixed-point types, platform macros
- `<SDL2/SDL.h>` ΓÇö SDL_RWops for cross-platform I/O
- `<vector>` ΓÇö std::vector for ID lists
- `<stdio.h>` ΓÇö Standard C I/O declarations
- **Forward declared**: FileSpecifier, LoadedResource (defined elsewhere)

# Source_Files/Files/tags.h
## File Purpose
Defines tags and typecodes for the WAD file format system used in Aleph One (a Marathon game engine). Tags serve as 4-character identifiers for different data structures (maps, objects, physics models, save game data), while typecodes identify file types at the OS level (scenarios, savegames, physics files, etc.). The typecode system abstracts platform-specific file type codes.

## Core Responsibilities
- Define `Typecode` enum for abstracting game file types (scenario, savegame, physics, shapes, sounds, images, preferences, music, etc.)
- Provide tag constants for WAD map structures (points, lines, sides, polygons, objects, light sources, platforms, terminals, etc.)
- Define tags for save/load game data (player, monsters, effects, projectiles, weapons, platforms, terminals, Lua state, etc.)
- Define tags for physics models (both current and Marathon 1 legacy versions)
- Define tags for embedded resources (shape patches, sound patches, scripts)
- Define tags for preferences storage (graphics, sound, network, input, environment, etc.)
- Manage platform-specific typecode mappings via accessor functions

## External Dependencies
- **cstypes.h** ΓÇô `uint32`, `FOUR_CHARS_TO_INT()` macro, `NONE` constant
- **filetypes_macintosh.c** ΓÇô Contains actual typecode value mappings (referenced in header comment, Feb 6, 2000)
- **\<vector\>** ΓÇô included but unused in this file

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

# Source_Files/Files/wad.h
## File Purpose
Defines the WAD (archive) file format structures and I/O functions for the Marathon/Aleph One game engine. WAD files serve as containers for game data (levels, assets, etc.), with version control and checksum integrity support. Provides both low-level file I/O and higher-level WAD data manipulation APIs.

## Core Responsibilities
- Define binary WAD file format (header, directory entries, data headers) with version evolution
- Read WAD files from disk and parse internal structures (headers, directories, tag data)
- Write and create WAD files with proper checksums and metadata
- Extract typed data from loaded WAD structures in memory
- Manage parent-child WAD relationships via checksums
- Provide flat serialization for data transfer and inflation back to in-memory form
- Memory lifecycle management for loaded WAD data

## External Dependencies
- `#include "tags.h"` ΓÇö WAD data type constants (`POINT_TAG`, `POLYGON_TAG`, `OBJECT_TAG`, etc.) and file typecode enums
- Forward declared: `FileSpecifier`, `OpenedFile` ΓÇö file abstraction layer (defined elsewhere, likely platform-specific)
- Standard: `uint32`, `int32`, `int16`, `byte` ΓÇö defined in `cstypes.h` (bundled with tags.h)
- Uses version constants `WADFILE_HAS_INFINITY_STUFF`, etc. for format evolution tracking

# Source_Files/Files/wad_prefs.cpp
## File Purpose
Provides persistent storage and retrieval of application preferences using WAD (Where's All the Data?) files. Wraps WAD file I/O with initialization, validation, and corruption recovery logic.

## Core Responsibilities
- Open/create preferences files with fallback recovery on corruption
- Retrieve preference entries, auto-initializing and validating them
- Write modified preferences back to disk with atomic file recreation
- Handle version mismatches and system errors gracefully

## External Dependencies
- **WAD I/O:** `create_empty_wad()`, `open_wad_file_for_reading()`, `open_wad_file_for_writing()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `write_wad_header()`, `write_wad()`, `write_directorys()`, `extract_type_from_wad()`, `append_data_to_wad()`, `calculate_wad_length()`, `close_wad_file()`, `free_wad()` (all defined elsewhere, likely `wad.cpp`).
- **File abstraction:** `FileSpecifier` (OO file spec; methods: `SetToPreferencesDir()`, `Exists()`, `Create()`, `Delete()`, `GetError()`, `GetType()`), `OpenedFile` (opaque file handle).
- **Error system:** `error_pending()`, `get_game_error()`, `set_game_error()` (defined in `game_errors`).
- **Standard library:** `<cstring>`, `<cstdlib>`, `<stdexcept>` for memory and string utilities.

# Source_Files/Files/wad_prefs.h
## File Purpose
Header for a preferences management system that reads and writes game configuration data using the WAD file format. Provides initialization, validation, and serialization of typed preference data through callback-based registration.

## Core Responsibilities
- Open and manage preferences files (FileSpecifier-based)
- Load typed preference data with custom initialization and validation callbacks
- Persist preferences back to the WAD file
- Maintain internal structure linking file specifier to WAD data

## External Dependencies
- **FileHandler.h:** `FileSpecifier` class for file/directory abstraction
- **wad.h:** `wad_data` struct, `WadDataType` typedef, WAD file I/O functions (`read_indexed_wad_from_file`, `free_wad`, etc.)
- **tags.h** (via wad.h): Type code constants (`Typecode`)

# Source_Files/Files/wad_sdl.cpp
## File Purpose
Implements SDL-based map file (WAD) discovery with support for both local filesystem and Steam Workshop sources. Provides two search strategies: by file checksum and by modification date.

## Core Responsibilities
- Locate WAD/map files by verifying checksums stored at file offset 0x44
- Find files matching specific modification dates
- Delegate to `FileFinder` base class for recursive directory traversal
- Integrate Steam Workshop item search with local data path search
- Convert game typecodes to Steam Workshop item types

## External Dependencies
- **FileHandler.h**: `FileSpecifier`, `OpenedFile`, `DirectorySpecifier` (file abstraction layer)
- **find_files.h**: `FileFinder` (base class for search implementations)
- **SDL2/SDL_endian.h**: `SDL_ReadBE32()` for endian-safe checksum reading
- **steamshim_child.h**: `ItemType` enum, `item_subscribed_query_result::item` struct (Steam integration, conditional)
- **cseries.h**: Standard includes, type definitions

# Source_Files/Files/WadImageCache.cpp
## File Purpose
Implements an on-disk LRU cache for thumbnail images extracted from WAD files. Manages image resizing, storage with UUID-based filenames, and persistence of cache metadata to an INI file with automatic eviction when a size limit is exceeded.

## Core Responsibilities
- Extract images from WAD files and resize them to requested dimensions
- Cache resized images on disk with auto-generated UUID filenames
- Maintain LRU (Least Recently Used) ordering and evict entries when cache size exceeds a configurable limit
- Load/save cache metadata (file paths, checksums, dimensions, sizes) to `Cache.ini`
- Provide fast O(1) lookup of cached images via dual data structure (list + map)
- Support optional autosave on cache modifications
- Handle image format conversions (PNG if available, fallback to BMP)

## External Dependencies

- **SDL2:** `SDL_Surface`, `SDL_RWops`, `SDL_LoadBMP_RW`, `SDL_SaveBMP`; conditionally `SDL2/SDL_image.h` for `IMG_Load_RW`, `IMG_SavePNG`
- **Boost:** UUID generation (`boost/uuid/uuid.hpp`, `boost/uuid/uuid_generators.hpp`, `boost/uuid/uuid_io.hpp`)
- **FileSpecifier** (from `FileHandler.h`): file I/O, path management, temp files
- **WadImageDescriptor, WAD functions** (from `wad.h`): `open_wad_file_for_reading`, `read_wad_header`, `read_indexed_wad_from_file`, `extract_type_from_wad`, `free_wad`, `close_wad_file`, `WadDataType`
- **InfoTree** (from `InfoTree.h`): INI file parsing and serialization
- **Game error system** (`game_errors.h`): `clear_game_error()`
- **SDL_Resize** (`sdl_resize.h`): Lanczos-filtered image resizing
- **Logging** (`Logging.h`): `logError` macro

# Source_Files/Files/WadImageCache.h
## File Purpose
Provides an on-disk LRU cache for thumbnail images stored in WAD files, enabling fast retrieval of resized images without re-decoding from the WAD archive. Manages cache persistence, size limits, and access tracking.

## Core Responsibilities
- Singleton instance management for global cache access
- Loading and caching thumbnail images at arbitrary resize dimensions
- LRU eviction when cache size exceeds configured limit
- Persistent cache metadata storage (filenames, sizes) to disk
- Image resizing and format conversion (SDL_Surface)
- Cache validation and expiration based on WAD file checksums

## External Dependencies
- **FileHandler.h:** `FileSpecifier` for WAD file paths and operations
- **wad.h:** `WadDataType` enum/typedef for image tags in WAD archives
- **SDL2 (SDL_Surface):** Image buffer format and manipulation
- **Standard Library:** `<tuple>`, `<list>`, `<map>`, `<string>` (STL containers)
- **Not directly visible:** WAD I/O functions (likely in wad.cpp or separate WAD reader); disk I/O for cache files


