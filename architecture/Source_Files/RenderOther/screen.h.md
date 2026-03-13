# Source_Files/RenderOther/screen.h

## File Purpose
Header for the Aleph One game engine's screen and rendering management system. Defines the singleton `Screen` class for mode/viewport management and declares functions for HUD rendering, color management, gamma correction, screen effects, and script-accessible display elements.

## Core Responsibilities
- Screen mode detection, selection, and dimension queries
- Viewport and clipping rectangle management for 3D view, HUD, map, and terminal
- Color table management (world, visible, interface) and gamma correction
- Screen effects (teleport, extravision, tunnel vision)
- HUD rendering control and script-based HUD element management
- Fullscreen toggle and display mode changes
- Overhead map zoom and visibility control
- Screenshot/screen dump functionality
- Main screen rendering pipeline and surface/window management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Screen` | class | Singleton managing all screen state, modes, and viewport rects |
| `screen_mode_data` | struct (fwd decl) | Display mode configuration (defined elsewhere) |
| `color_table` | struct (extern) | Color palette data for rendering (defined elsewhere) |
| `SDL_Rect` | struct (SDL2) | Rectangle for viewport/clipping regions |
| `SDL_PixelFormat` | struct (SDL2) | Pixel format descriptor |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `pixel_format_16` | SDL_PixelFormat | extern | 16-bit pixel format descriptor |
| `pixel_format_32` | SDL_PixelFormat | extern | 32-bit pixel format descriptor |
| `world_color_table` | color_table* | extern | Active game world palette |
| `visible_color_table` | color_table* | extern | Currently visible/rendered palette |
| `interface_color_table` | color_table* | extern | UI/menu palette |

## Key Functions / Methods

### Screen::instance()
- Signature: `static inline Screen* instance()`
- Purpose: Access the singleton Screen instance
- Outputs: Pointer to static Screen instance
- Notes: Singleton pattern; always returns valid non-null pointer

### Screen::Initialize()
- Signature: `void Initialize(screen_mode_data* mode)`
- Purpose: Initialize screen subsystem with given display mode
- Inputs: Display mode configuration
- Side effects: Initializes SDL surfaces/windows, sets m_initialized flag

### Screen::FindMode() / ModeWidth() / ModeHeight()
- Purpose: Query available display modes and lookup dimensions by index
- Inputs: Width/height (FindMode), mode index (dimension accessors)
- Outputs: Mode index or -1 if not found; width/height in pixels
- Notes: Linear search; dimension accessors assume valid index

### Screen accessors
Methods like `height()`, `width()`, `pixel_scale()`, `openGL()`, `hud()`, etc.
- Purpose: Query current screen state (dimensions, scale, rendering backend, HUD status)
- Outputs: Integer dimensions, float scale, or boolean flags
- Notes: Return current active display configuration

### Screen viewport/rect methods
`window_rect()`, `view_rect()`, `map_rect()`, `term_rect()`, `hud_rect()`, `OpenGLViewPort()`
- Purpose: Retrieve SDL_Rect regions for different screen areas
- Outputs: SDL_Rect defining viewport boundaries
- Notes: Used by rendering code to partition display between 3D view, UI, map, terminal

### Screen::bound_screen() / bound_screen_to_rect()
- Signature: `void bound_screen(bool in_game = true)`, `void bound_screen_to_rect(SDL_Rect &r, bool in_game = true)`
- Purpose: Set OpenGL scissor/clipping to screen bounds or specific rect
- Inputs: Optional in_game flag to select world vs. menu rendering context
- Side effects: Modifies OpenGL scissor state

### render_screen()
- Signature: `void render_screen(short ticks_elapsed)`
- Purpose: Main rendering entry point; frames the 3D view, HUD, map, and UI
- Inputs: Elapsed time in ticks for animation/frame-rate-independent updates
- Side effects: Renders to active SDL surface/OpenGL context

### Script HUD functions
`SetScriptHUDText()`, `SetScriptHUDColor()`, `SetScriptHUDIcon()`, `SetScriptHUDSquare()`, `IsScriptHUDNonlocal()`, `SetScriptHUDNonlocal()`
- Purpose: Allow Lua scripts to customize HUD overlays (up to 6 elements per player)
- Inputs: Player index, element index (0ΓÇô5), text/color/icon data
- Side effects: Update HUD overlay state; trigger redraw
- Notes: Color uses terminal palette; icon format undocumented; NULL/empty string removes element

### change_screen_mode()
- Signature: `void change_screen_mode(struct screen_mode_data *mode, bool redraw, bool resize_hud = false)`, `void change_screen_mode(short screentype)`
- Purpose: Switch display mode or screen type (level, menu, chapter)
- Inputs: Mode data or screen-type enum; flags to control redraw and HUD resize
- Side effects: Reinitializes screen, updates viewports, may resize HUD

### MainScreen* functions
`MainScreenVisible()`, `MainScreenLogicalWidth()`, `MainScreenLogicalHeight()`, `MainScreenWindowWidth()`, `MainScreenWindowHeight()`, `MainScreenPixelWidth()`, `MainScreenPixelHeight()`, `MainScreenPixelScale()`, `MainScreenIsOpenGL()`, `MainScreenSwap()`, `MainScreenCenterMouse()`, `MainScreenWindow()`, `MainScreenSurface()`, `MainScreenUpdateRect()`, `MainScreenUpdateRects()`
- Purpose: Query and control the main SDL window/surface (dimensions in logical, window, and pixel space; backend type; buffer swaps; mouse centering; rect updates)
- Outputs: Dimensions, booleans, SDL_Window/SDL_Surface pointers
- Side effects: `Swap()` flips buffers; `UpdateRect(s)()` marks regions dirty

### Other functions
- `toggle_overhead_map_display_status()`, `zoom_overhead_map_in/out()` ΓÇö map UI control
- `start_teleporting_effect()`, `start_extravision_effect()` ΓÇö screen effects
- `change_screen_clut()`, `animate_screen_clut()`, `change_gamma_level()` ΓÇö palette and gamma
- `toggle_fullscreen()` ΓÇö display mode toggle
- `dump_screen()`, `reset_screen()`, `enter_screen()`, `exit_screen()` ΓÇö lifecycle and debugging
- `GetTunnelVision()`, `SetTunnelVision()` ΓÇö tunnel vision mode
- `RequestDrawingHUD()`, `RequestDrawingTerm()`, `draw_intro_screen()` ΓÇö render requests

## Control Flow Notes
This is the rendering subsystem entry point. `render_screen(ticks_elapsed)` is called each frame from the main loop; it composites the 3D view (via OpenGL or software), overlays the HUD, map, and terminal, applies color/gamma, and swaps buffers (via `MainScreenSwap()`). Script HUD elements are drawn on top. Screen mode changes are handled reactively via `change_screen_mode()` and `toggle_fullscreen()`. Effects (teleport, extravision) are triggered by gameplay events but rendered here.

## External Dependencies
- **SDL2**: `SDL.h` for window, surface, rect, pixel format, events
- **Standard library**: `<utility>`, `<vector>` for mode list and pairs
- **Internal types** (defined elsewhere):
  - `struct Rect` (forward declared; likely a simple rect type)
  - `struct screen_mode_data` (display mode config)
  - `struct color_table` (palette data)
- **OpenGL** (implied by `OpenGLViewPort()`, `openGL()` queries, scissor methods, but not explicitly included here)
