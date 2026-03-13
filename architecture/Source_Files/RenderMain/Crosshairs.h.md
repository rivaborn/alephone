# Source_Files/RenderMain/Crosshairs.h

## File Purpose
Interface header for crosshair rendering and configuration in the game engine. Defines the data structure for crosshair properties (color, thickness, shape, opacity) and provides functions for state management, configuration, and rendering to an SDL surface.

## Core Responsibilities
- Define `CrosshairData` struct encapsulating crosshair visual properties and rendering state
- Provide configuration dialog interface for player customization
- Manage crosshair active/inactive state
- Render crosshairs to the backbuffer
- Expose access to stored crosshair preferences

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CrosshairData` | struct | Holds all crosshair parameters: color (RGB), thickness, distance from center, line length, shape mode (real crosshairs vs circle), opacity, and pre-calculated OpenGL color values |
| `CHShape_RealCrosshairs`, `CHShape_Circle` | enum constants | Shape modes for crosshair rendering |

## Global / File-Static State
None.

## Key Functions / Methods

### Configure_Crosshairs
- **Signature:** `bool Configure_Crosshairs(CrosshairData &Data)`
- **Purpose:** Display a configuration dialog allowing the player to customize crosshair appearance
- **Inputs:** Reference to a `CrosshairData` struct to be modified
- **Outputs/Return:** `true` if user confirmed changes, `false` if canceled (struct unchanged on cancel)
- **Side effects:** Modal dialog display; updates crosshair configuration if confirmed
- **Calls:** (implementation in PlayerDialogs.c)
- **Notes:** Modifies the passed struct in-place on success

### GetCrosshairData
- **Signature:** `CrosshairData& GetCrosshairData()`
- **Purpose:** Retrieve the active crosshair configuration from stored preferences
- **Inputs:** None
- **Outputs/Return:** Reference to the crosshair data loaded from preferences
- **Side effects:** None (retrieval only)
- **Calls:** (implementation in preferences.c)
- **Notes:** Returns a reference, allowing callers to query current settings

### Crosshairs_IsActive
- **Signature:** `bool Crosshairs_IsActive()`
- **Purpose:** Query whether crosshairs are currently enabled for rendering
- **Inputs:** None
- **Outputs/Return:** `true` if active, `false` if inactive
- **Calls:** (implementation not in this file)

### Crosshairs_SetActive
- **Signature:** `bool Crosshairs_SetActive(bool NewState)`
- **Purpose:** Enable or disable crosshair rendering
- **Inputs:** `NewState` ΓÇô desired active state
- **Outputs/Return:** Previous active state
- **Calls:** (implementation not in this file)

### Crosshairs_Render
- **Signature:** `bool Crosshairs_Render(SDL_Surface *s)`
- **Purpose:** Draw the crosshair onto the given SDL surface
- **Inputs:** Pointer to target SDL surface (typically backbuffer)
- **Outputs/Return:** Status (true: rendered successfully, false: inactive or error)
- **Side effects:** Modifies surface pixels; likely reads global crosshair state and configuration
- **Calls:** (implementation not in this file)

## Control Flow Notes
Crosshairs are a rendering feature invoked during the HUD/UI render phase. Configuration occurs outside the main loop (player settings). Active state and rendering are queried/executed per-frame during render time.

## External Dependencies
- **cseries.h** ΓÇô Provides `RGBColor` struct (three uint16 components: red, green, blue) and platform abstraction
- **SDL2/SDL.h** ΓÇô Imported indirectly through cseries.h; `SDL_Surface` used as rendering target
