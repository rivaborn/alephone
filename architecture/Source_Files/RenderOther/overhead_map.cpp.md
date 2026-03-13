# Source_Files/RenderOther/overhead_map.cpp

## File Purpose
Implements the overhead map (automap/motion sensor) rendering system for the Marathon game engine. Manages configuration, font initialization, and delegates rendering to platform-specific implementations (SDL or OpenGL).

## Core Responsibilities
- Initialize and manage overhead map rendering configuration with default colors, line styles, and entity definitions
- Lazy-initialize fonts for map annotations and titles
- Route rendering calls to appropriate backend (software SDL or OpenGL) based on `OGL_MapActive` flag
- Reset automap visibility data based on current mode (Normal, CurrentlyVisible, or All)
- Parse and apply XML configuration for overhead map display parameters (colors, fonts, visibilities)
- Maintain MML (Marathon Markup Language) configuration state for reset/reload cycles

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OvhdMap_CfgDataStruct` | struct | Complete overhead map configuration: polygon colors (by terrain type), line definitions, thing display styles, monster-to-display-type mappings, player entity size/shape, annotation fonts, path colors, and visibility flags |
| `OverheadMap_SDL_Class` | class | Software renderer backend using SDL |
| `OverheadMap_OGL_Class` | class | OpenGL renderer backend |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `OvhdMap_ConfigData` | `OvhdMap_CfgDataStruct` | static | Current overhead map rendering configuration; initialized with game defaults (terrain colors, line styles, entity sizes) |
| `MapFontsInited` | bool | static | Flag to track one-time font initialization |
| `OGL_MapActive` | bool | global | Controls rendering backend selection; set externally to toggle between OpenGL and software rendering |
| `OverheadMap_SW` | `OverheadMap_SDL_Class` | static | SDL/software renderer instance |
| `OverheadMap_OGL` | `OverheadMap_OGL_Class` | static | OpenGL renderer instance (conditional on `HAVE_OPENGL`) |
| `OverheadMapMode` | short | static | Current map display mode: Normal, CurrentlyVisible, or All |
| `original_OvhdMap_ConfigData` | `OvhdMap_CfgDataStruct` | static | Backup configuration for MML reset |
| `original_OverheadMapMode` | short | static | Backup mode for MML reset |

## Key Functions / Methods

### InitMapFonts
- **Signature:** `static void InitMapFonts()`
- **Purpose:** Lazy-initialize all fonts used for rendering map annotations and the map title.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies `MapFontsInited` flag; calls `Init()` on each font in `OvhdMap_ConfigData` annotation definitions and map name data.
- **Calls:** Loops through `annotation_definitions` and calls `Font.Init()` on fonts not yet initialized.
- **Notes:** Called once per rendering session; font initialization is deferred to first render to avoid overhead at startup.

### _render_overhead_map
- **Signature:** `void _render_overhead_map(struct overhead_map_data *data)`
- **Purpose:** Main entry point for rendering the overhead map; initializes fonts and dispatches to the appropriate renderer backend.
- **Inputs:** `overhead_map_data *data` ΓÇö view parameters (mode, scale, origin, dimensions, visibility flags)
- **Outputs/Return:** None
- **Side effects:** May initialize fonts; sets `ConfigPtr` on the selected renderer; dispatches rendering call to either OpenGL or SDL renderer.
- **Calls:** `InitMapFonts()`, renderer's `Render(*data)` method.
- **Notes:** Backend selection is determined by `OGL_MapActive` at call time; allows mixed rendering (e.g., main view in OpenGL, checkpoint view in software).

### ResetOverheadMap
- **Signature:** `void ResetOverheadMap()`
- **Purpose:** Reset automap visibility based on the current rendering mode.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears or sets bits in `automap_lines` and `automap_polygons` bitsets.
- **Calls:** `memset()`
- **Notes:** Behavior depends on `OverheadMapMode`: Normal (no change), CurrentlyVisible (clear visibility), All (mark all visible).

### reset_mml_overhead_map
- **Signature:** `void reset_mml_overhead_map()`
- **Purpose:** Restore overhead map configuration and mode to original/default values.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Overwrites `OvhdMap_ConfigData` and `OverheadMapMode` with backed-up originals.
- **Calls:** None (simple assignment)

### parse_mml_overhead_map
- **Signature:** `void parse_mml_overhead_map(const InfoTree& root)`
- **Purpose:** Parse XML configuration elements for overhead map display settings.
- **Inputs:** `InfoTree& root` ΓÇö XML tree node containing overhead map settings
- **Outputs/Return:** None
- **Side effects:** Updates `OvhdMap_ConfigData` fields and `OverheadMapMode` based on XML attributes and child nodes.
- **Calls:** `root.read_indexed()`, `root.read_attr()`, `root.read_attr_bounded()`, `root.read_color()`, `root.read_font()`, loop over `root.children_named()`.
- **Notes:** Handles nested XML elements for: mode, title offset, monster type assignments, visibility toggles (aliens/items/projectiles/paths), line widths, colors (indexed by type), and fonts (indexed by annotation size or map name).

## Control Flow Notes
- **Initialization:** `OvhdMap_ConfigData` is statically initialized with default terrain colors, line styles, entity definitions, and font specifications.
- **XML Configuration:** `parse_mml_overhead_map()` is called at startup (via Marathon's MML pipeline) to override defaults; `reset_mml_overhead_map()` allows reload cycles.
- **Rendering:** `_render_overhead_map()` is called during each frame's HUD/automap draw phase; it dispatches to the active backend (SDL or OpenGL).
- **Font Lifecycle:** Fonts are initialized on first render (`InitMapFonts()`) to defer overhead; initialization is cached via `MapFontsInited` flag.

## External Dependencies
- **Notable includes:** `cseries.h` (platform utilities), `shell.h` (`_get_player_color`), `map.h` (map constants, `dynamic_world`), `monsters.h` (monster type enums), `OverheadMap_SDL.h`, `OverheadMap_OGL.h` (renderer implementations), `InfoTree.h` (XML parsing)
- **Defined elsewhere:** `automap_lines`, `automap_polygons`, `dynamic_world`, `NUMBER_OF_MONSTER_TYPES`, `NUMBER_OF_COLLECTIONS`, `OVERHEAD_MAP_MAXIMUM_SCALE`, `OVERHEAD_MAP_MINIMUM_SCALE`, `OverheadMapClass`, `OverheadMap_SDL_Class`, `OverheadMap_OGL_Class`
