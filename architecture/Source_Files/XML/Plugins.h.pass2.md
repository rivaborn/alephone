# Source_Files/XML/Plugins.h - Enhanced Analysis

## Architectural Role

Plugins.h is the **runtime configuration and extensibility hub** for Aleph One, sitting at the intersection of files, XML/MML loading, and game subsystems. It abstracts the discovery, validation, and mode-aware activation of all plugin-provided content (Lua scripts, MML configurations, graphics/audio patches, map resource overrides), and exposes a simple query interface (`get_resource`, `find_*_lua`) that downstream subsystems (rendering, sound, Lua) call to retrieve plugin overrides. The singleton pattern ensures all subsystems observe the same plugin state across the engine's lifecycle.

## Key Cross-References

### Incoming (who depends on this file)
- **XML/MML subsystem** (e.g., `XML_MakeRoot.cpp`): calls `Plugins::load_mml()` to parse MML from enabled plugins during config initialization
- **Rendering pipeline** (RenderMain, OGL_*): calls `Plugins::load_shapes_patches()` and `Plugin::get_resource()` to load graphics overrides
- **Sound subsystem** (SoundManager, Music): calls `Plugins::load_sounds_patches()` to register audio overrides
- **Files/GameWorld** (game_wad.cpp, map loading): calls `Plugins::set_map_checksum()` and `get_resource()` to apply map-specific resource patches
- **Lua subsystem**: calls `Plugins::find_solo_lua()`, `find_hud_lua()`, `find_stats_lua()` to locate script providers and enforce `SoloLuaWriteAccess` exclusivity rules
- **Shell/Interface** (initialization): calls `enumerate()`, `set_mode()`, `enable()`/`disable()` to discover plugins and toggle them before game start

### Outgoing (what this file depends on)
- **FileHandler.h**: uses `DirectorySpecifier` for plugin paths, `LoadedResource` for resource I/O abstraction
- **boost::filesystem**: path manipulation for `enable()`/`disable()` operations
- **STL containers**: `std::vector<Plugin>` for plugin list, `std::map<resource_key_t, std::string>` for map patch lookups, `std::stack<ScopedSearchPath>` for scoped directory search overrides

## Design Patterns & Rationale

| Pattern | Implementation | Why |
|---------|---|---|
| **Singleton** | `static Plugins* instance()` | Single global plugin state; all subsystems must observe consistent enable/disable state and checksums |
| **Resource Manager** | `enumerate()` + validation cache (`m_validated`) | Lazy validation; plugins discovered once at startup, re-validated only on mode changes or explicit invalidation |
| **Strategy (Game Mode)** | `set_mode(GameMode)` + conditional loading | Solo mode excludes multiplayer Lua; Menu mode skips in-game scripts; reduces memory and prevents incompatible resource mixing |
| **Bit Flags** | `SoloLuaWriteAccess` (0x01, 0x02, 0x04, ...) | Efficient exclusivity enforcement; `world` flag claims global Lua monopoly; `fog`/`music`/`overlays` compete only with their type |
| **RAII Directory Stack** | `m_search_paths` (stack of `ScopedSearchPath`) | Safe scoped path management; plugins discovered via stacked search paths that unwind on scope exit |

**Rationale:** The plugin system balances extensibility (plugins can override almost any resource) with safety (checksums, validation, exclusive access). Mode-aware loading reduces memory and prevents runtime conflicts. The search path stack allows temporary plugin directories (e.g., temporary workshop staging) without global state pollution.

## Data Flow Through This File

1. **Initialization Phase:**
   - `enumerate()` ΓåÆ scans disk via search paths ΓåÆ populates `m_plugins` vector with metadata (name, version, required_version, mmls, lua scripts, patches)
   - Validation deferred (lazy) via `m_validated` flag

2. **Mode & Enable/Disable Phase:**
   - `set_mode(GameMode)` changes filter (Menu/Solo/Net) ΓåÆ `invalidate()` ΓåÆ triggers re-validation
   - `enable(path)` / `disable(path)` toggle `Plugin::enabled` ΓåÆ `invalidate()`

3. **Content Loading Phase (per game mode):**
   - `load_mml(bool load_menu_mml_only)` ΓåÆ walks enabled plugins ΓåÆ yields MML paths to XML_MakeRoot
   - `load_shapes_patches(bool opengl)` ΓåÆ filters `ShapesPatch::requires_opengl` ΓåÆ registers overrides in rendering subsystem
   - `load_sounds_patches()` ΓåÆ registers sound overrides
   - `find_*_lua()` ΓåÆ returns plugin(s) providing HUD/solo/stats Lua with `SoloLuaWriteAccess` constraints enforced

4. **Map-Specific Overrides:**
   - `set_map_checksum(uint32_t)` stores map ID
   - `get_resource(type, id, rsrc)` ΓåÆ checks all plugins' `MapPatch::parent_checksums` ΓåÆ if match, looks up in `resource_map` ΓåÆ fills `LoadedResource`

## Learning Notes

1. **Aleph One Extensibility Model:** Unlike modern engines with explicit plugin APIs, Aleph One treats plugins as resource bundles ΓÇö MML for config, Lua for logic, file patches for assets. This pre-dates structured plugin frameworks; it's more akin to mod systems in Doom/Marathon.

2. **Solo Mode Access Control:** The `SoloLuaWriteAccess` bit flags reveal a design decision: solo Lua can claim exclusive access (world, fog, music, overlays) to prevent conflicts, but ephemera and sound allow multiples. This suggests solo mode was designed to allow per-map Lua customization without plugin interference.

3. **Checksum-Based Resource Patching:** Map patches keyed by checksum (rather than map name) exploit WAD file integrity checksums as unique identifiers. This allows plugins to target specific map versions without brittle name-based matching.

4. **RAII Search Paths:** The `std::stack<ScopedSearchPath>` pattern is uncommon in modern code (prefer dependency injection). It reflects an older C++ era where global state scoping was the primary tool for context management.

5. **Deferred Validation:** Plugins are discovered eagerly but validated lazily (on demand, cached). This balances startup performance with safe discovery.

## Potential Issues

1. **Thread Safety:** No visible synchronization on `m_plugins`, `m_validated`, `m_mode`, or `m_map_checksum`. If any of these are accessed from multiple threads (e.g., Lua thread + main thread), races are possible. The `PluginLoader` friend class may manage this externally.

2. **Search Path Stack Fragility:** If a `ScopedSearchPath` guard exits unexpectedly (exception in plugin loading), the stack may unwind prematurely, leaving plugins inaccessible. No guard against incomplete unwinding.

3. **No Rollback on Partial Load Failure:** If `load_shapes_patches()` fails halfway, no cleanup is visible. Half-loaded patches could remain in the rendering subsystem, causing visual corruption.

4. **Resource Leak in `get_resource()`:** The `LoadedResource` output param is filled but never validated as successfully written. If a plugin's resource file is missing, the caller must detect an empty `rsrc`.
