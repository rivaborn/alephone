# Source_Files/RenderMain/OGL_Render.h

## File Purpose
OpenGL interface header providing functions to integrate OpenGL 3D rendering with the Marathon (Aleph One) game engine. Declares the public API for managing rendering context, setting view parameters, and rendering geometric primitives, sprites, text, and UI elements.

## Core Responsibilities
- **Context lifecycle**: Initialize/destroy OpenGL rendering context and manage GPU resources
- **View configuration**: Set perspective parameters, camera position, and projection matrices
- **Geometric rendering**: Render walls, sprites, and other game objects
- **UI rendering**: Display text, crosshairs, cursors, rectangles, and overhead map elements
- **State management**: Query rendering status (active, 2D enabled, current fog) and control rendering modes
- **Rendering modes**: Support foreground (weapons-in-hand) and main view rendering with separate view parameters
- **Buffer management**: Handle window bounds, back buffer allocation, and buffer swapping

## Key Types / Data Structures
None defined in this file. Uses types from included headers:

| Name | Kind | Purpose |
|------|------|---------|
| `view_data` | struct | Camera/projection parameters, view state (render.h) |
| `polygon_definition` | struct | Wall geometry and texturing (render.h) |
| `rectangle_definition` | struct | Sprite/billboard geometry (render.h) |
| `OGL_FogData` | struct | Fog configuration (OGL_Setup.h) |
| `Rect` | struct | Rectangle bounds (platform) |
| `SDL_Rect` | struct | SDL rectangle (SDL) |
| `world_point2d` | struct | 2D world coordinates (world.h) |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_StartRun
- Signature: `bool OGL_StartRun()`
- Purpose: Create and initialize OpenGL rendering context
- Inputs: None
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Allocates GPU resources, establishes rendering context
- Calls: Not inferable from header
- Notes: Must be called once at engine startup; paired with OGL_StopRun

### OGL_StopRun
- Signature: `bool OGL_StopRun()`
- Purpose: Destroy OpenGL rendering context and release resources
- Inputs: None
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Deallocates GPU resources, destroys context
- Calls: Not inferable from header
- Notes: Cleanup operation; called at shutdown

### OGL_SetWindow
- Signature: `bool OGL_SetWindow(Rect &ScreenBounds, Rect &ViewBounds, bool UseBackBuffer)`
- Purpose: Configure OpenGL rendering window and viewport
- Inputs: Screen rectangle, view rectangle, back buffer flag
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Updates GPU viewport, allocates/deallocates back buffer
- Calls: Not inferable from header
- Notes: Called when window resizes or rendering surface changes

### OGL_SetView
- Signature: `bool OGL_SetView(view_data &View)`
- Purpose: Set camera projection and view matrices for perspective rendering
- Inputs: `view_data` struct with FOV, position, orientation, screen dimensions
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Updates GPU projection/view matrices
- Calls: Not inferable from header
- Notes: Called each frame before main rendering pass

### OGL_RenderWall
- Signature: `bool OGL_RenderWall(polygon_definition& RenderPolygon, bool IsVertical)`
- Purpose: Render a wall polygon with texture and lighting
- Inputs: Polygon geometry/texture; flag for vertical vs. horizontal
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Draws to GPU framebuffer
- Calls: Not inferable from header
- Notes: Called multiple times per frame during main render pass

### OGL_RenderSprite
- Signature: `bool OGL_RenderSprite(rectangle_definition& RenderRectangle)`
- Purpose: Render a sprite (actor, item, effect) as billboard geometry
- Inputs: Sprite geometry and texture definition
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Draws to GPU framebuffer
- Calls: Not inferable from header
- Notes: Used for dynamic objects and visual effects

### OGL_RenderText
- Signature: `bool OGL_RenderText(short BaseX, short BaseY, const char *Text, unsigned char r, g, b)`
- Purpose: Render ASCII text string at screen position with specified color
- Inputs: Screen coords, C string, RGB values (default white)
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Draws text glyphs to GPU framebuffer
- Calls: Not inferable from header
- Notes: Used for HUD messages, console, menus; `OGL_TextWidth` queries text dimensions

### OGL_SwapBuffers
- Signature: `bool OGL_SwapBuffers()`
- Purpose: Swap front/back buffers to display rendered frame
- Inputs: None
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Display update; completes frame presentation
- Calls: Not inferable from header
- Notes: Called once per frame after all rendering complete

**Notes**: Other functions (`OGL_SetForeground`, `OGL_SetForegroundView`, `OGL_RenderRect`, `OGL_RenderLines`, `OGL_RenderCrosshairs`) provide similar entry points for HUD rendering, geometric primitives, and UI elements.

## Control Flow Notes
Typical per-frame sequence:
1. **OGL_SetView** ΓÇô configure projection
2. **OGL_StartMain** ΓÇô begin scene
3. **OGL_RenderWall** (├ùN) ΓÇô walls, polygons
4. **OGL_RenderSprite** (├ùN) ΓÇô actors, effects
5. **OGL_EndMain** ΓÇô end scene
6. **OGL_SetForeground** ΓÇô switch to HUD view
7. **OGL_RenderText**, **OGL_RenderCrosshairs**, **OGL_RenderRect** (├ùN) ΓÇô UI/HUD
8. **OGL_SwapBuffers** ΓÇô display frame

## External Dependencies
- **OGL_Setup.h** ΓÇô OpenGL configuration, `OGL_FogData`, color types
- **render.h** ΓÇô `view_data`, `polygon_definition`, `rectangle_definition`
- Implicit: SDL (SDL_Rect), platform rectangles (Rect)
- Implicit: OpenGL headers (called from .cpp implementation, not visible here)
