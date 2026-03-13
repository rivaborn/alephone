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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Plugin` | struct | Single plugin descriptor: directory, metadata, resource lists, enabled/override flags |
| `SoloLuaWriteAccess` | class | Bitflags for solo Lua write permissions (world, fog, music, overlays, ephemera, sound); conflict detection |
| `ScenarioInfo` | struct | Scenario version/name/ID for compatibility requirements |
| `ShapesPatch` | struct | Shape patch metadata: file path, OpenGL requirement flag |
| `MapPatch` | struct | Map-specific resources: parent map checksums, resource type/IDΓåÆpath mapping |
| `Plugins` | class | Singleton manager: plugin list, validation cache, game mode, search path stack |
| `PluginLoader` | class | XML parser helper; friend class with write access to Plugins::add() |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `static Plugins*` | singleton | Lazy-initialized plugin manager instance |
| (namespace alias) | `algo` | namespace | `boost::algorithm` shorthand for string predicates |

## Key Functions / Methods

### `Plugin::compatible()`
- Signature: `bool compatible() const`
- Purpose: Check if plugin satisfies engine version and current scenario requirements
- Inputs: (implicit) required_version, required_scenarios
- Outputs: `true` if version ΓëÑ A1_DATE_VERSION and scenario matches (or no requirements)
- Calls: `Scenario::instance()->GetName/ID/Version()` (defined elsewhere)

### `Plugin::allowed()`
- Signature: `bool allowed() const`
- Purpose: Determine if stats_lua content is permitted by network preferences
- Outputs: `false` if stats_lua exists and `!network_preferences->allow_stats`
- Notes: Empty stats_lua always returns true

### `Plugin::valid()`
- Signature: `bool valid() const`
- Purpose: Check if plugin is active (enabled, not overridden, respects environment prefs)
- Outputs: `true` only if enabled and not marked overridden (set by Plugins::validate)
- Notes: Considers `environment_preferences->use_solo_lua` and game mode

### `Plugin::get_resource()`
- Signature: `bool get_resource(uint32_t checksum, uint32_t type, int id, LoadedResource& rsrc) const`
- Purpose: Retrieve map-specific resource patch if parent map checksum matches
- Inputs: map checksum, resource type code, resource ID
- Outputs: `LoadedResource` populated with malloc'd file data; `true` on success
- Calls: `file.SetNameWithPath()`, `file.Open()`, `ofile.Read()`, `rsrc.SetData()` (FileHandler API)
- Side effects: Allocates memory (caller must release via LoadedResource destructor)
- Notes: Iterates map_patches in reverse order (last wins)

### `Plugins::instance()`
- Signature: `static Plugins* instance()`
- Purpose: Lazy singleton accessor
- Outputs: Pointer to global Plugins manager
- Side effects: Creates instance on first call

### `Plugins::disable()` / `Plugins::enable()`
- Signature: `bool disable/enable(const boost::filesystem::path& path)`
- Purpose: Toggle plugin enabled flag by directory path; invalidate validation cache
- Outputs: `true` if plugin found and toggled
- Side effects: Sets `m_validated = false`

### `Plugins::enumerate()`
- Signature: `void enumerate()`
- Purpose: Discover all plugins from data search paths and Steam workshop
- Side effects: Populates `m_plugins` vector; clears game errors; sets `m_validated = false`
- Calls: `PluginLoader::ParseDirectory()` for each path
- Notes: Iterates `data_search_path` (defined elsewhere) + Steam subscribed items

### `Plugins::load_mml()`
- Signature: `void load_mml(bool load_menu_mml_only)`
- Purpose: Load MML modifications from all valid, active plugins
- Calls: `validate()`, then `load_mmls()` helper for each valid plugin ΓåÆ `ParseMMLFromFile()`
- Side effects: MML modifications applied (defined elsewhere)

### `Plugins::load_shapes_patches()` / `load_sounds_patches()`
- Signature: `void load_shapes_patches(bool is_opengl)` / `void load_sounds_patches()`
- Purpose: Load shape and sound patch files from valid plugins
- Calls: `validate()`, then `load_shapes_patch()` / `sounds_patches.add()` (defined elsewhere)
- Side effects: Patches registered with resource loaders
- Notes: Shape patches respect `requires_opengl` flag

### `Plugins::find_hud_lua()` / `find_solo_lua()` / `find_stats_lua()` / `find_theme()`
- Signature: `const Plugin* find_{hud,stats,theme}_lua()` / `std::vector<const Plugin*> find_solo_lua()`
- Purpose: Locate plugin owning exclusive Lua/theme resource
- Outputs: Pointer to plugin (or nullptr); solo_lua returns vector
- Calls: `validate()`
- Notes: Reverse iteration (last plugin in list takes precedence); only returns valid, enabled plugins

### `PluginLoader::ParsePlugin()`
- Signature: `bool ParsePlugin(FileSpecifier& file_name)`
- Purpose: Parse Plugin.xml and create Plugin struct; add to manager if valid
- Inputs: FileSpecifier pointing to Plugin.xml
- Outputs: `true` if file opened (parsing errors logged but don't fail)
- Side effects: Allocates Plugin, calls `Plugins::instance()->add()`; logs XML/file errors
- Calls: `InfoTree::load_xml()`, `plugin_file_exists()`, `utf8_to_int()` helper
- Notes: 
  - Validates file existence for all referenced resources
  - Enforces scenario name/ID/version length limits
  - Rejects multiple `<solo_lua>` tags; legacy `solo_lua` attribute for backwards compat
  - Theme plugins clear incompatible resources (hud_lua, solo_lua, shapes, sounds, maps)
  - Parses write_access flags for solo Lua

### `PluginLoader::ParseDirectory()`
- Signature: `bool ParseDirectory(FileSpecifier& dir)`
- Purpose: Recursively search directory (and ZIP files) for Plugin.xml; parse each found
- Inputs: Directory path
- Outputs: `false` if ReadDirectory fails; `true` otherwise
- Calls: `ParsePlugin()` recursively; `file.ReadZIP()` for ZIP archives
- Notes: Skips hidden files (dot-prefix); searches inside .zip/.ZIP for Plugin.xml

### `Plugins::validate()`
- Signature: `void validate()`
- Purpose: Compute active plugin set, enforcing exclusivity for HUD Lua, stats Lua, themes, and write-access-exclusive solo Lua
- Side effects: Sets `overridden` and `overridden_solo` flags on plugins; caches result (`m_validated = true`)
- Notes:
  - Two passes: one tracking solo Lua (with write-access conflict check), one excluding it
  - Reverse iteration: later plugins override earlier ones
  - Solo Lua write access flags create mutual exclusion:
    - `world` flag excludes all other Lua
    - `fog`, `music`, `overlays` exclude same-type Lua
    - `ephemera`, `sound` allow multiples
  - Checks enabled, compatible, and allowed state before marking override

### `Plugins::get_resource()` / `set_map_checksum()`
- Signature: `bool get_resource(uint32_t type, int id, LoadedResource& rsrc)` / `void set_map_checksum(uint32_t checksum)`
- Purpose: Fetch map-specific patched resource; manage search path for patch discovery
- Inputs: resource type/ID; map checksum
- Side effects: `set_map_checksum()` prepends matching plugin directories to search path stack
- Notes: Validates plugins enabled & compatible before searching

## Control Flow Notes
1. **Initialization**: `enumerate()` scans all paths, populates `m_plugins`, sets `m_validated = false`
2. **Validation**: `validate()` called lazily before any resource load, resolves conflicts, caches with `m_validated`
3. **Script Finding**: `find_*_lua()` / `find_theme()` return active script provider (via reverse iteration)
4. **Resource Loading**: `load_mml()`, `load_shapes_patches()`, `load_sounds_patches()` iterate valid plugins
5. **Map Patches**: `set_map_checksum()` configures search path; `get_resource()` queries patches per-map

## External Dependencies
- **FileHandler.h**: `FileSpecifier`, `DirectorySpecifier`, `OpenedFile`, `LoadedResource`, `ScopedSearchPath`
- **Logging.h**: `logWarning()`, `logError()`, `logContext()`
- **InfoTree.h**: XML tree parsing (`InfoTree::load_xml()`, `children_named()`, `read_attr()`)
- **preferences.h**: `network_preferences->allow_stats`, `environment_preferences->use_solo_lua`
- **Scenario.h**: `Scenario::instance()->GetName/ID/Version()`
- **alephversion.h**: `A1_DATE_VERSION` for version checks
- **boost/algorithm/string/predicate.hpp**: `ends_with()` for ZIP detection
- **Defined elsewhere**: `ParseMMLFromFile()`, `load_shapes_patch()`, `Scenario` singleton, global `data_search_path`, `subscribed_workshop_items` (Steam)
