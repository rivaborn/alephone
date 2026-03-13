# Source_Files/shell.h

## File Purpose
Main application shell header for the Aleph One game engine (Marathon remake). Declares the interface for application lifecycle, display configuration, shape rendering, color management, and input device handling.

## Core Responsibilities
- Define display/graphics configuration structure (`screen_mode_data`)
- Declare application initialization, shutdown, and main event loop
- Declare shape rendering and color lookup functions
- Declare MML script loading for game configuration
- Declare path expansion/contraction utilities for resource loading
- Define input device types and key constants
- Declare debug/UI text output functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `screen_mode_data` | struct | Holds display settings: resolution, fullscreen, bit depth, gamma, HUD scale, bobbing type, FOV |
| `BobbingType` | enum class | Camera/weapon bobbing modes: none, camera_and_weapon, weapon_only |

## Global / File-Static State
None.

## Key Functions / Methods

### global_idle_proc
- **Signature:** `void global_idle_proc(void)`
- **Purpose:** Called during idle periods to perform background processing
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Likely updates application state or renders frames
- **Calls:** Not inferable from this file
- **Notes:** Presumably called frequently from main event loop

### LoadBaseMMLScripts
- **Signature:** `void LoadBaseMMLScripts(bool load_menu_mml_only)`
- **Purpose:** Loads base MML (Aleph One markup) scripts for game configuration
- **Inputs:** `load_menu_mml_only` ΓÇô if true, loads only menu-related scripts
- **Outputs/Return:** None
- **Side effects:** Initializes game configuration state
- **Calls:** Not inferable from this file
- **Notes:** Called during initialization

### get_shape_surface
- **Signature:** `SDL_Surface *get_shape_surface(int shape, int collection = NONE, byte** outPointerToPixelData = NULL, float inIllumination = -1.0f, bool inShrinkImage = false)`
- **Purpose:** Retrieves and renders a game shape (sprite/bitmap) to an SDL surface
- **Inputs:** Shape index, collection ID, optional illumination/darkness level, shrink flag
- **Outputs/Return:** `SDL_Surface*` to rendered shape; optionally populates `outPointerToPixelData` for RLE-compressed shapes
- **Side effects:** Allocates SDL surface; caller must free
- **Calls:** Not inferable from this file
- **Notes:** Supports RLE shape decompression, team/player color tinting via illumination, quarter-scale shrinking for RLE shapes only

### expand_symbolic_paths / contract_symbolic_paths
- **Signature:** `char *expand_symbolic_paths(char *dest, const char *src, int maxlen)` / `contract_symbolic_paths(...)`
- **Purpose:** Convert paths between symbolic (e.g., `$APP_DIR`) and absolute forms
- **Inputs:** Destination buffer, source path, max length
- **Outputs/Return:** Pointer to destination buffer
- **Side effects:** Modifies destination buffer
- **Calls:** Not inferable from this file

### _get_player_color / _get_interface_color
- **Signature:** `void _get_player_color(size_t color_index, RGBColor *color)` and SDL_Color variant
- **Purpose:** Look up player or interface color by index
- **Inputs:** Color index, output structure
- **Outputs/Return:** Populates provided color structure
- **Side effects:** None
- **Notes:** Overloaded for both RGBColor and SDL_Color

### main_event_loop
- **Signature:** `void main_event_loop(void)`
- **Purpose:** Main game/application event loop
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Runs until application shutdown
- **Calls:** Not inferable from this file

### initialize_application / shutdown_application
- **Signature:** `void initialize_application(void)` / `void shutdown_application(void)`
- **Purpose:** Application startup and cleanup
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initialize/tear down all engine systems
- **Calls:** Not inferable from this file

### handle_open_document
- **Signature:** `bool handle_open_document(const std::string& filename)`
- **Purpose:** Handle file open requests (e.g., save game, replay, document load)
- **Inputs:** Filename to open
- **Outputs/Return:** Success/failure boolean
- **Side effects:** May load game state or media
- **Calls:** Not inferable from this file

## Control Flow Notes
Application initialization flow: `initialize_application()` ΓåÆ `LoadBaseMMLScripts()` ΓåÆ `load_environment_from_preferences()` ΓåÆ `main_event_loop()` which calls `global_idle_proc()` repeatedly ΓåÆ `shutdown_application()`. Shape rendering and color lookups are utilities available throughout the main loop.

## External Dependencies
- **Includes:** `cstypes.h` (basic integer types), `<string>` (std::string)
- **Forward declarations:** `FileSpecifier`, `RGBColor`, `SDL_Color`, `SDL_Surface` (defined elsewhere)
- **SDL2 integration:** Uses SDL surfaces and colors for cross-platform rendering
