# Source_Files/XML/InfoTree.cpp
## File Purpose
Implements serialization and deserialization for game configuration data using Boost.PropertyTree, supporting XML and INI formats. Provides conversion utilities between file representations and game-specific types (colors, shapes, damage, fonts, paths, angles).

## Core Responsibilities
- Load/save XML and INI configuration files from `FileSpecifier` paths or streams
- Convert between game-specific numeric types and tree representation (fixed-point, world units, angles)
- Encode/decode paths (symbolic path expansion/contraction) and strings (Mac Roman Γåö UTF-8)
- Serialize/deserialize structured game data (colors, shapes, damage definitions, fonts)
- Provide iteration over named child nodes with range adaptor

## External Dependencies
- **Boost.PropertyTree** (`<boost/property_tree/*.hpp>`) ΓÇô core tree structure and XML/INI parsers
- **Boost.Iostreams** (`<boost/iostreams/stream.hpp>`) ΓÇô stream abstraction for file I/O
- **Boost.Range** (`<boost/range/adaptor/map.hpp>`) ΓÇô range adaptor for map-like iteration
- **Game engine utilities** ΓÇô `FileHandler.h` (`FileSpecifier`, `opened_file_device`), `FontHandler.h` (`FontSpecifier`), `TextStrings.h` (`DeUTF8_C`, `mac_roman_to_utf8`), `shell.h` (`expand_symbolic_paths`, `contract_symbolic_paths`)
- **Game data types** (defined elsewhere) ΓÇô `_fixed`, `angle`, `shape_descriptor`, `damage_definition`, `RGBColor`, `rgb_color`

# Source_Files/XML/InfoTree.h
## File Purpose

InfoTree wraps Boost's PropertyTree (iptree) to provide a type-safe, exception-safe API for reading and writing game configuration data from XML and INI files. It specializes in parsing Aleph One game world types (colors, shapes, damage definitions, fonts, angles) from structured data, serving as a bridge between file I/O and the game engine's data model.

## Core Responsibilities

- Load/save XML and INI configuration files via Boost PropertyTree
- Provide type-safe template readers with fallback defaults for arbitrary types
- Parse game-specific types: colors, shape descriptors, damage definitions, fonts, fixed-point angles
- Read XML attributes with bounds checking and enum validation
- Handle attribute paths with proper XML namespace prefixes (`<xmlattr>.`)
- Write attributes and game-specific data back to tree structures
- Expose named child element iteration via `children_named()`

## External Dependencies

- `<boost/property_tree/ptree.hpp>`, `<boost/property_tree/ini_parser.hpp>`, `<boost/property_tree/xml_parser.hpp>` ΓÇö XML/INI parsing and serialization
- `<boost/range/any_range.hpp>` ΓÇö Range adaptation for child iteration
- `<type_traits>` ΓÇö SFINAE overloading for enum vs. non-enum read_attr_bounded
- `FileHandler.h` ΓÇö FileSpecifier for abstract file paths
- `FontHandler.h` ΓÇö FontSpecifier type for font configuration
- `map.h`, `world.h` ΓÇö Game world types (shape_descriptor, damage_definition, angle, _fixed, RGBColor)
- `cseries.h` ΓÇö Base types and macros (RGBColor struct, NONE constant)

# Source_Files/XML/Plugins.cpp
## File Purpose
Plugin manager for the Aleph One game engine that discovers, validates, and loads plugins from XML metadata. Enforces compatibility rules, resolves resource conflicts, and manages exclusive plugin types (HUD Lua, themes, stats Lua).

## Core Responsibilities
- **Plugin discovery & enumeration**: Scans directories and Steam workshop for Plugin.xml files; supports ZIP archives
- **XML parsing & metadata extraction**: Parses plugin descriptors, scenario requirements, resource paths, and Lua script specs
- **Validation & conflict resolution**: Enforces mutual exclusivity (only one HUD Lua, theme, or stats Lua active); respects version/scenario compatibility
- **Resource loading**: Loads MML, shape patches, sound patches, and map-specific resources from valid plugins
- **Script management**: Locates HUD Lua, solo Lua (with write access control), and stats Lua scripts
- **Plugin state**: Enable/disable by path; invalidation cache for re-validation

## External Dependencies
- **FileHandler.h**: `FileSpecifier`, `DirectorySpecifier`, `OpenedFile`, `LoadedResource`, `ScopedSearchPath`
- **Logging.h**: `logWarning()`, `logError()`, `logContext()`
- **InfoTree.h**: XML tree parsing (`InfoTree::load_xml()`, `children_named()`, `read_attr()`)
- **preferences.h**: `network_preferences->allow_stats`, `environment_preferences->use_solo_lua`
- **Scenario.h**: `Scenario::instance()->GetName/ID/Version()`
- **alephversion.h**: `A1_DATE_VERSION` for version checks
- **boost/algorithm/string/predicate.hpp**: `ends_with()` for ZIP detection
- **Defined elsewhere**: `ParseMMLFromFile()`, `load_shapes_patch()`, `Scenario` singleton, global `data_search_path`, `subscribed_workshop_items` (Steam)

# Source_Files/XML/Plugins.h
## File Purpose
Plugin manager for Aleph One game engine. Defines plugin metadata structures, access control for Lua scripts, and a singleton manager for discovering, validating, loading, and enabling/disabling plugins and their associated resources (shapes, sounds, maps, Lua scripts).

## Core Responsibilities
- Plugin discovery and enumeration from directories
- Plugin metadata parsing and validation (version, compatibility, requirements)
- Game mode management (Menu, Solo, Network) with mode-specific resource loading
- Lua script access control and exclusivity enforcement (via `SoloLuaWriteAccess`)
- Plugin enable/disable state management
- Resource patch loading (shapes, sounds, maps) with checksum-based lookups
- Plugin compatibility checks and resource queries

## External Dependencies
- **FileHandler.h**: `DirectorySpecifier`, `LoadedResource` for file/resource I/O abstractions
- **boost::filesystem**: Path manipulation (enable/disable by path)
- **STL**: `<list>`, `<map>`, `<set>`, `<stack>`, `<vector>`, `<string>`

# Source_Files/XML/QuickSave.cpp
## File Purpose
Implements a manager for auto-named saved games with metadata and preview images. Handles creation, loading, deletion, and UI dialogs for quick save management in the Aleph One game engine.

## Core Responsibilities
- Create, load, and delete quick save files
- Generate and cache map preview images for saves
- Provide dialog UI for selecting and managing saves
- Store and retrieve save metadata (level name, playtime, player count)
- Enumerate saves from the quick-saves directory
- Maintain LRU cache of preview images to limit memory usage

## External Dependencies

- **FileHandler.h** ΓÇô `FileSpecifier`, `OpenedFile`, directory operations
- **world.h** ΓÇô `world_point2d`, `world_point3d`, coordinate types
- **map.h** ΓÇô `TICKS_PER_MINUTE`, `TICKS_PER_SECOND`, `local_player`, `dynamic_world`, `static_world`
- **wad.h** ΓÇô WAD file I/O (`read_wad_header()`, `read_indexed_wad_from_file()`, `write_wad()`, etc.)
- **overhead_map.h** ΓÇô `_render_overhead_map()`, `overhead_map_data`, rendering modes
- **screen_drawing.h** ΓÇô `_set_port_to_custom()`, `_restore_port()`, `draw_text()`
- **sdl_dialogs.h, sdl_widgets.h** ΓÇô Dialog and widget base classes
- **WadImageCache.h** ΓÇô `WadImageCache::instance()`, `WadImageDescriptor`
- **InfoTree.h** ΓÇô `InfoTree` for metadata parsing/generation
- **SDL2/SDL_image.h** ΓÇô `IMG_SavePNG_RW()` (conditional)
- **boost/algorithm/string/** ΓÇô String manipulation (replace, ends_with)
- Defined elsewhere: `OGL_MapActive` (flag), `environment_preferences->maximum_quick_saves`, various `_typecode_*` and tag constants

# Source_Files/XML/QuickSave.h
## File Purpose
Defines a manager for auto-named saved game files ("quick saves") in the Aleph One game engine. Provides structures to represent individual saves and a singleton class to enumerate, manage, and manipulate collections of quick saves.

## Core Responsibilities
- Define `QuickSave` struct to encapsulate metadata (file path, name, level, timestamp, tick count, player count)
- Provide `QuickSaves` singleton to maintain an in-memory collection of discovered quick saves
- Support enumeration of saves from disk, deletion of surplus saves, and iteration over the collection
- Expose global functions for creating, deleting, and loading quick saves
- Track networked save status for multiplayer games

## External Dependencies
- **FileHandler.h**: `FileSpecifier` class for file/path abstraction
- **Standard library**: `<string>`, `<vector>`, `<time.h>` (for `time_t`)
- Implicit: Definition of `QuickSaveLoader` (friend class, defined elsewhere)

# Source_Files/XML/XML_LevelScript.cpp
## File Purpose
Manages XML-based scripts embedded in map files for the Aleph One game engine. Orchestrates per-level execution of MML scripts, Lua scripts, music playlists, movies, and load screens. Supports special pseudo-levels for defaults, restoration, and end-of-game sequences.

## Core Responsibilities
- Load and parse `marathon_levels` XML from map file resource 128
- Maintain a registry of level scripts indexed by level number
- Execute appropriate scripts when levels are loaded or game ends
- Manage movie specifications and playback parameters per level
- Coordinate music playlist setup and random-order playback
- Configure load screen appearance with images, colors, and progress indicators
- Support pseudo-levels (Default, Restore, End) for common operations
- Handle embedded MML and Lua script chunks from binary data

## External Dependencies
- **Music.h:** `Music::instance()` for playlist management, fading, playback
- **XML_ParseTreeRoot.h:** `ParseMMLFromData()` to parse MML from memory
- **lua_script.h:** `LoadLuaScript()` to load Lua scripts (with `_embedded_lua_script` context)
- **OGL_LoadScreen.h:** `OGL_LoadScreen::instance()` for load screen rendering (OpenGL)
- **Plugins.h:** `Plugins::instance()->load_mml()` for plugin scripts
- **InfoTree.h:** XML parsing infrastructure (`InfoTree::load_xml()`, node iteration, attribute reading)
- **AStream.h:** `AIStreamBE` for big-endian binary stream parsing
- **Logging.h:** `logError()` for parse error reporting
- **images.h:** `get_text_resource_from_scenario()` to retrieve TEXT resource 128 from map file

# Source_Files/XML/XML_LevelScript.h
## File Purpose
Header file declaring the interface for loading, parsing, and executing XML-based level scripts in Aleph One. Manages level initialization scripts written in Pfhortran/MML (Marathon Markup Language) and Lua, handles level and end-game movie playback, and provides restoration of default parameters across level transitions.

## Core Responsibilities
- Loading level scripts from map file resources (resource 128)
- Running level-specific MML/Pfhortran scripts during level initialization
- Managing embedded MML and Lua script data (get/set operations)
- Locating and retrieving movie files for level playback and end-game sequences
- Executing end-of-game scripts and restoration scripts for parameter reset
- Parsing and managing MML default level configurations

## External Dependencies
- **FileHandler.h** ΓÇö `FileSpecifier` class for file path abstraction and I/O
- **Implicit:** Pfhortran/MML interpreter (not declared here; likely in implementation file)
- **Forward-declared:** `InfoTree` class for XML configuration parsing (defined elsewhere)
- **C standard:** `<cstdint>` types (`uint8`, `size_t`); `<cstdio>` indirectly via FileHandler

# Source_Files/XML/XML_MakeRoot.cpp
## File Purpose
Root XML parser dispatcher for Marathon's modular XML configuration system (MML). Loads and validates Marathon XML files, then routes parsed elements to appropriate subsystem parsers. Provides a centralized reset mechanism for all MML-configured values.

## Core Responsibilities
- Reset all MML-modified game subsystems to hardcoded defaults
- Load XML from files or memory buffers with comprehensive error handling
- Traverse XML tree and dispatch child elements to ~30 subsystem-specific parsers
- Support conditional parsing (menu-only vs. full) based on context
- Log parse errors with file path and exception details

## External Dependencies
- **Headers:** cseries.h (platform/base), 30+ subsystem headers (interface.h, weapons.h, monsters.h, items.h, etc.), InfoTree.h
- **External symbols:** InfoTree class, FileSpecifier class, ~60 parse_mml_*() and reset_mml_*() functions (defined elsewhere), logError() function

# Source_Files/XML/XML_ParseTreeRoot.h
## File Purpose
Declares the root element of the XML parse tree for the Aleph One game engine and provides the public interface for parsing Marathon Markup Language (MML) from files and in-memory buffers. This is the entry point for MML configuration loading during engine initialization.

## Core Responsibilities
- Export the public API for MML parsing (`ParseMMLFromFile`, `ParseMMLFromData`)
- Declare the configuration reset function (`ResetAllMMLValues`)
- Define the absolute root element containing all valid XML document trees
- Support both file-based and buffer-based MML parsing

## External Dependencies
- `FileSpecifier` (forward declaration; defined elsewhere in engine)
- Standard library: `<stddef.h>` (for `size_t`)
- Comment references GPL 3.0 licensing


