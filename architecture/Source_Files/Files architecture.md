# Subsystem Overview

## Purpose
Manages all file I/O operations for the Aleph One game engine, including cross-platform file abstraction, WAD (Where's All the Data) archive format handling, game save/load serialization, resource fork emulation for macOS compatibility, and physics data import. Provides binary serialization with endian conversion and integrity checksums.

## Key Files
| File | Role |
|------|------|
| FileHandler.h/cpp | Cross-platform file I/O abstraction via SDL_RWops; FileSpecifier path management; native file dialogs (NFD or SDL fallback); ZIP archive support |
| wad.h/cpp | WAD file format reader/writer (versions 0ΓÇô4); multi-version binary parsing; tag-based data extraction; CRC validation |
| AStream.h/cpp | Templated big-endian/little-endian binary serialization streams with bounds checking and exception handling |
| Packing.h/cpp | Serialization primitives for 16/32-bit integer pack/unpack with endianness conversion |
| game_wad.h/cpp | Game state persistence: map loading, level initialization, save game creation/restoration, network map sync |
| resource_manager.h/cpp | macOS resource fork emulation; AppleSingle/MacBinary parsing; resource lookup by type+ID or index |
| tags.h | Typecode enum and WAD tag constants for file types and data structures |
| crc.h/cpp | CRC-32 and CCITT CRC-16 computation for files and buffers; data integrity verification |
| import_definitions.cpp | Physics definition unpacking (monsters, weapons, effects, projectiles) from Marathon 1 or WAD formats |
| wad_prefs.h/cpp | Preferences persistence via WAD file format with corruption recovery |
| find_files.h / find_files_sdl.cpp | Recursive directory traversal framework with typecode-based filtering |
| WadImageCache.h/cpp | On-disk LRU thumbnail cache for WAD images; metadata persistence to INI; size-limit eviction |
| wad_sdl.cpp | WAD/map file discovery by checksum or modification date; Steam Workshop integration |
| preprocess_map_sdl.cpp | Default resource file location; save game dialogs and operations |

## Core Responsibilities
- Cross-platform file I/O abstraction (open, read, write, seek, close, metadata) wrapping SDL_RWops
- WAD archive format reading/writing with version evolution (Marathon 1 through Infinity compatibility)
- Game state serialization: save/load with full world state (player, monsters, projectiles, effects, platforms, terminals, Lua state)
- Binary pack/unpack with big-endian and little-endian byte order conversion and alignment handling
- macOS resource fork emulation via AppleSingle/MacBinary format detection and parsing on non-Mac platforms
- Physics definition import and unpacking for monsters, weapons, effects, projectiles, and constants
- CRC-32 and CRC-16 checksum calculation and validation for file integrity
- File discovery via recursive traversal with typecode-based filtering, including Steam Workshop sources
- Preferences persistence with WAD format backing, corruption recovery, and validation
- Image caching with LRU eviction, resize-on-demand, and metadata persistence
- Resource type/ID lookup across multiple open resource files with stack-based context

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- FileHandler: open/close/read/write operations, directory enumeration, file dialogs, path management
- WAD I/O: `open_wad_file_for_reading()`, `read_indexed_wad_from_file()`, `write_wad()`, `extract_type_from_wad()` for map/save data extraction
- Game save/load: `save_game()`, `load_game()`, `load_map_wad()` for persistence
- Physics import: `import_physics_models()`, `parse_physics_wad()` for entity definitions
- Preferences: `read_wad_prefs()`, `write_wad_prefs()` for configuration storage
- Image caching: `WadImageCache::Get()` for thumbnail retrieval with auto-resize

**Consumes from other subsystems:**
- GameWorld (map.h, world.h): all geometry and entity structures for save serialization
- Game entities (monsters.h, projectiles.h, effects.h, weapons.h, platforms.h, player.h): dynamic state packing for save files
- Network (network.h): map checksum distribution, player data for netgames
- Scripting (XML_LevelScript.h): MML level script loading from WAD
- UI/Shell (interface.h, shell.h, sdl_dialogs.h): error reporting, dialogs, resource strings
- Audio (SoundManager.h): dialog sound effects
- Rendering (screen.h, images.h): render system context, HUD images

## Runtime Role
- **Initialization**: Locate and validate default resources (maps, physics, shapes, sounds) via `have_default_files()`; load preferences from WAD; initialize physics definitions
- **Level transitions**: Load map geometry and entities via `load_map_wad()`; save/restore game state via `save_game()` / `load_game()`
- **Frame**: Image cache access on demand via `WadImageCache` for UI rendering
- **Shutdown**: Flush preferences to disk via `write_wad_prefs()`; cleanup resource file stack and cache metadata

## Notable Implementation Details
- WAD format supports version evolution (constants `WADFILE_HAS_INFINITY_STUFF`, etc.) for Marathon 1ΓÇôInfinity compatibility
- Binary serialization uses templated `basic_astream<T>` for compile-time endian selection (big-endian default, little-endian override) vs. legacy `Packing.h` macros
- Resource fork emulation detects format (AppleSingle, MacBinary II/III, raw fork) transparently on parse
- Image cache uses UUID-based filenames and dual data structure (doubly-linked list + map) for O(1) LRU eviction
- Map discovery supports both local filesystem and Steam Workshop via checksum lookup at file offset 0x44
- Level-transition malloc (`level_transition_malloc()`) used for map data allocation to enable deallocation on level exit
- Preference entries auto-initialize and validate; corruption triggers fallback to default WAD creation
