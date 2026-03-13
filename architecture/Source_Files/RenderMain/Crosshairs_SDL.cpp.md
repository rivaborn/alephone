# Source_Files/RenderMain/Crosshairs_SDL.cpp

## File Purpose
Implements SDL-based rendering of the game's crosshair HUD element in two configurable shapes. Manages crosshair visibility state and draws crosshairs to an SDL surface each frame, or delegates to Lua HUD if enabled.

## Core Responsibilities
- Maintain crosshair active/inactive state via file-static variable
- Render crosshairs in two shape modes: standard (4 rectangles) or circular (octagon with line segments)
- Map crosshair color data from 16-bit RGB to SDL pixel format
- Calculate crosshair geometry relative to surface center with configurable dimensions
- Check Lua HUD override flag before rendering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CrosshairData` | struct (external) | Holds color, thickness, distance-from-center, length, shape type, opacity, and precomputed GL color |
| `SDL_Surface` | opaque type (SDL2) | Target surface for pixel rendering |
| `world_point2d` | struct (external) | 2D coordinate pair for line segment endpoints |
| `RGBColor` | struct (external) | 16-bit per-channel RGB color |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_Crosshairs_IsActive` | `bool` | static (file) | Tracks whether crosshair rendering is enabled |
| `use_lua_hud_crosshairs` | `bool` | extern | Flag indicating Lua HUD system overrides crosshairs |

## Key Functions / Methods

### Crosshairs_IsActive
- **Signature:** `bool Crosshairs_IsActive(void)`
- **Purpose:** Query current crosshair visibility state
- **Inputs:** None
- **Outputs/Return:** `bool` ΓÇö true if crosshairs active, false otherwise
- **Side effects:** None
- **Calls:** None
- **Notes:** Simple getter for internal state

### Crosshairs_SetActive
- **Signature:** `bool Crosshairs_SetActive(bool NewState)`
- **Purpose:** Change crosshair visibility state
- **Inputs:** `NewState` ΓÇö desired active/inactive state
- **Outputs/Return:** `bool` ΓÇö the new state (for chaining)
- **Side effects:** Modifies `_Crosshairs_IsActive`
- **Calls:** None
- **Notes:** Assignment operator returns the assigned value

### Crosshairs_Render
- **Signature:** `bool Crosshairs_Render(SDL_Surface *s)`
- **Purpose:** Draw crosshairs to the provided SDL surface each frame
- **Inputs:** `s` ΓÇö SDL surface to render onto (typically the HUD buffer)
- **Outputs/Return:** `bool` ΓÇö true if rendered, false if skipped (inactive or Lua override)
- **Side effects:** Pixels written to SDL surface; no allocations
- **Calls:** `GetCrosshairData()`, `SDL_MapRGB()`, `SDL_FillRect()`, `draw_line()`
- **Notes:**
  - Early exit if `_Crosshairs_IsActive` is false or Lua HUD is active
  - Calculates surface center as `(w/2 - 1, h/2 - 1)` 
  - **RealCrosshairs mode:** Draws 4 axis-aligned rectangles (left, right, top, bottom) at configurable distance and thickness
  - **Circle mode:** Approximates circle as octagon with 12 line segments (3 per quadrant: vertical, diagonal, horizontal). Precalculates 6 x-coordinates and 6 y-coordinates via half-width recursion for the octagon vertices

## Control Flow Notes
Appears to be called during HUD rendering pipeline, once per frame, after the main 3D world has been drawn. Early-exit checks allow Lua scripts to fully override crosshair rendering. Octagon rendering uses nested loops (2├ù2 quadrants) to avoid coordinate repetition.

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_MapRGB()`, `SDL_FillRect()`
- **Crosshairs.h:** `CrosshairData` struct, `GetCrosshairData()` function
- **screen_drawing.h:** `draw_line()` function (for octagon edges)
- **world.h:** `world_point2d` structure
- **cseries.h:** Base types (`uint32`, `RGBColor`)
