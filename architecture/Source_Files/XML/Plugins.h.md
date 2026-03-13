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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ScenarioInfo` | struct | Stores scenario metadata (name, ID, version) required by a plugin |
| `ShapesPatch` | struct | Defines graphics patch with OpenGL requirement flag and file path |
| `SoloLuaWriteAccess` | class | Bit-flag access control for Lua scripts (world, fog, music, overlays, ephemera, sound) with exclusivity rules |
| `MapPatch` | struct | Maps parent checksums to resource replacements (type/ID ΓåÆ filepath) |
| `Plugin` | struct | Plugin descriptor with directory, metadata, Lua scripts, resource patches, enabled/override state |
| `Plugins` | class | Singleton manager for plugin collection, enumeration, validation, loading, and queries |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Plugins::m_plugins` | `std::vector<Plugin>` | static (singleton) | Collection of discovered plugins |
| `Plugins::m_validated` | `bool` | static (singleton) | Cache invalidation flag; triggers re-validation on `invalidate()` |
| `Plugins::m_mode` | `GameMode` enum | static (singleton) | Current game context (Menu, Solo, Network) |
| `Plugins::m_search_paths` | `std::stack<ScopedSearchPath>` | static (singleton) | Directory search path stack for resource lookups |
| `Plugins::m_map_checksum` | `uint32_t` | static (singleton) | Current map checksum for resource matching in patches |

## Key Functions / Methods

### Plugins::instance
- Signature: `static Plugins* instance()`
- Purpose: Returns singleton instance of plugin manager
- Outputs/Return: Pointer to global Plugins object
- Notes: Singleton pattern; no public constructor

### Plugins::enumerate
- Signature: `void enumerate()`
- Purpose: Discovers and loads all plugins from disk; populates `m_plugins` vector
- Inputs: None (uses internal search paths)
- Side effects: Populates `m_plugins`; may invoke `invalidate()` logic
- Notes: Called at initialization; does not load resources yet

### Plugins::load_mml
- Signature: `void load_mml(bool load_menu_mml_only)`
- Purpose: Loads MML (Marathon Markup Language) files from all enabled plugins
- Inputs: `load_menu_mml_only` ΓÇô if true, skips non-menu MML files for speed
- Side effects: Modifies engine configuration via MML parsing

### Plugins::load_shapes_patches / load_sounds_patches
- Signature: `void load_shapes_patches(bool opengl)`; `void load_sounds_patches()`
- Purpose: Loads graphics and audio patches from enabled plugins, applying overrides
- Inputs: `opengl` flag for shapes filter (if true, only load patches marked OpenGL-compatible)
- Side effects: Registers resource overrides in engine resource system

### Plugins::enable / disable
- Signature: `bool enable(const boost::filesystem::path& path)`; `bool disable(...)`
- Purpose: Toggles plugin active state by path
- Inputs: Boost filesystem path to plugin directory
- Outputs/Return: true if operation succeeded
- Notes: May trigger re-validation; plugins can be marked override-only

### Plugins::find_hud_lua / find_solo_lua / find_stats_lua / find_theme
- Signature: `const Plugin* find_hud_lua()`; `std::vector<const Plugin*> find_solo_lua()`; etc.
- Purpose: Searches enabled plugins for specific resource types (HUD script, solo/coop Lua, stats, theme)
- Outputs/Return: Pointer(s) to plugin(s) providing the resource, or nullptr if none
- Notes: `find_solo_lua()` returns vector to enforce multiple-plugin support with `SoloLuaWriteAccess` exclusivity rules

### Plugins::get_resource
- Signature: `bool get_resource(uint32_t type, int id, LoadedResource& rsrc)`
- Purpose: Queries all enabled plugins' map patches to find a resource override matching map checksum
- Inputs: Resource type, ID, and current `m_map_checksum`
- Outputs/Return: true if resource found and loaded into `rsrc`; false otherwise
- Calls: `Plugin::get_resource()` on each plugin; delegates to `MapPatch` matching

### Plugins::set_map_checksum
- Signature: `void set_map_checksum(uint32_t checksum)`
- Purpose: Updates current map checksum; used to filter `MapPatch` lookups by parent map
- Inputs: 32-bit checksum of loaded map
- Side effects: Modifies `m_map_checksum`

### SoloLuaWriteAccess::is_excluded
- Signature: `bool is_excluded(uint32_t flags) const`
- Purpose: Checks if the given flags conflict with this access object's exclusive rights
- Inputs: Flags bitmask (world, fog, music, overlays, etc.)
- Outputs/Return: true if flags are mutually exclusive and already claimed
- Notes: Enforces single-provider rule for world/fog/music/overlays; allows ephemera and sound multiples

### Plugin validation methods
- `bool compatible()`, `bool allowed()`, `bool valid()`
- Purpose: Check version compatibility, access permissions, and overall validity
- Notes: `compatible()` likely checks `required_version` against engine version; `allowed()` checks scenario requirements

## Control Flow Notes
**Initialization**: `enumerate()` ΓåÆ `validate()` (on demand, cached) ΓåÆ internal state ready.

**Frame/Load Cycle**: When map loads, `set_map_checksum()` is called; subsequent `get_resource()` queries use this checksum to find map-specific patch overrides.

**Game Mode Transitions**: `set_mode()` changes game context (Menu ΓåÆ Solo ΓåÆ Network); controls which MML and Lua scripts are loaded (e.g., solo-only Lua excluded in multiplayer).

**Resource Search Path**: `m_search_paths` stack allows scoped directory overrides (e.g., temporary plugin directories) via `ScopedSearchPath` RAII guards (see FileHandler.h).

## External Dependencies
- **FileHandler.h**: `DirectorySpecifier`, `LoadedResource` for file/resource I/O abstractions
- **boost::filesystem**: Path manipulation (enable/disable by path)
- **STL**: `<list>`, `<map>`, `<set>`, `<stack>`, `<vector>`, `<string>`
