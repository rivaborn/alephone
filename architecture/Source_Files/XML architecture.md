# Subsystem Overview

## Purpose
The XML subsystem provides centralized configuration, metadata, and scripting management for Aleph One. It parses Marathon Markup Language (MML) XML files to populate game subsystems, orchestrates level-specific scripts (MML, Lua, movies, music), manages plugins with resource patch loading, and handles quick save file metadata and previews. Serves as the bridge between file-based configuration and the runtime game engine.

## Key Files
| File | Role |
|------|------|
| InfoTree.cpp/h | Type-safe Boost.PropertyTree wrapper; XML/INI parsing and game-type serialization (colors, shapes, damage, fonts, angles) |
| XML_MakeRoot.cpp | Root MML dispatcher; loads XML, routes subsystem elements to ~60 parse_mml_*() handlers, provides reset mechanism |
| XML_ParseTreeRoot.h | Public MML API (`ParseMMLFromFile`, `ParseMMLFromData`, `ResetAllMMLValues`) |
| Plugins.cpp/h | Plugin discovery from directories/Steam workshop; validation, resource loading (MML, shapes, sounds, maps), exclusivity enforcement |
| XML_LevelScript.cpp/h | Per-level script orchestration; executes MML/Lua scripts, manages movies, music playlists, load screens from map resource 128 |
| QuickSave.cpp/h | Quick save file management; creation, metadata, preview image caching, UI dialogs, LRU cache for in-memory previews |

## Core Responsibilities
- Parse XML/INI configuration files via Boost.PropertyTree with bounds checking and enum validation
- Load and reset MML configurations during engine initialization and level transitions
- Route parsed XML elements to subsystem-specific parsers for weapons, monsters, items, physics, interface, etc.
- Discover and validate plugins from directories and Steam workshop with version/compatibility checks
- Load plugin resources (MML, shape patches, sound patches, map patches) and enforce mutual exclusivity (single HUD Lua, theme, stats Lua)
- Execute level-specific scripts, manage movie playback, coordinate music playlists, and configure load screens
- Manage quick save files with metadata (level name, playtime, player count) and preview images
- Serialize/deserialize game-specific numeric types (fixed-point angles, world units, colors) between XML and memory

## Key Interfaces & Data Flow
**Exposes:**
- `ParseMMLFromFile(FileSpecifier)` / `ParseMMLFromData(buffer, size)` ΓÇö entry points for MML loading
- `ResetAllMMLValues()` ΓÇö reset all subsystems to defaults
- `Plugins::instance()` ΓÇö singleton for plugin discovery, validation, resource loading
- `QuickSaves` ΓÇö singleton for enumerating, creating, deleting saved games
- `InfoTree::load_xml()`, `read_attr()`, `children_named()` ΓÇö XML parsing API

**Consumes:**
- **FileHandler**: `FileSpecifier`, `DirectorySpecifier`, `OpenedFile`, `LoadedResource` for abstract file I/O
- **GameWorld types**: `shape_descriptor`, `damage_definition`, `angle`, `_fixed`, `RGBColor`, `world_point2d/3d`
- **Subsystems**: ~60 `parse_mml_*()` / `reset_mml_*()` functions (weapons, monsters, items, interface, physics, etc.); Music, Lua, OGL_LoadScreen for level script execution
- **Boost libraries**: PropertyTree (XML/INI), Iostreams (streams), Range (iteration)
- **Scenario**: active map metadata and resource loading context
- **External**: Steam workshop API (`subscribed_workshop_items`); preferences (solo Lua, stats, network modes)

## Runtime Role
- **Initialization**: `ParseMMLFromFile()` called during engine startup to load default MML configurations and route elements to subsystems via dispatcher
- **Level load**: `XML_LevelScript` executes level-specific MML/Lua scripts, configures movies, music, and load screens from map resource 128
- **Per-frame**: Lua scripts (HUD, solo, stats) run via engine's Lua interpreter; music playlists advance
- **Plugin discovery**: Scanned on startup and when user accesses plugin UI
- **Shutdown**: No explicit unload; data persists until reset via `ResetAllMMLValues()`

## Notable Implementation Details
- **InfoTree namespace handling**: XML attributes accessed via `<xmlattr>.` prefix path
- **Encoding support**: Mac Roman Γåö UTF-8 conversion; symbolic path expansion/contraction
- **Plugin exclusivity**: Only one HUD Lua, theme, or stats Lua allowed active; validated during `load_mml()`
- **Steam integration**: `subscribed_workshop_items` provides workshop plugin enumeration; checksums for resource conflict resolution
- **Preview caching**: `WadImageCache` LRU for quick save preview images; `OGL_MapActive` determines software vs. OpenGL rendering
- **Pseudo-levels**: Default, Restore, End-of-game levels for common MML operations across all maps
- **Binary script embedding**: Level scripts parsed from big-endian binary data (`AIStreamBE`) in resource 128
