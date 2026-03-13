# Source_Files/RenderMain/Rasterizer_OGL.h

## File Purpose
OpenGL-specific implementation of the abstract `RasterizerClass` interface. Provides a thin adapter layer that delegates rendering operations to underlying OpenGL functions. Serves as a pluggable backend for the engine's rendering pipeline when `HAVE_OPENGL` is defined.

## Core Responsibilities
- Implement the rasterizer interface for OpenGL-based rendering
- Manage view setup and camera configuration for OpenGL
- Handle foreground layer rendering (weapons, hand-held objects, HUD elements)
- Delegate polygon (wall) and sprite rendering to OpenGL subsystem functions
- Act as a concrete factory/bridge between the abstract render interface and OGL_Render functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Rasterizer_OGL_Class | class | Concrete OpenGL rasterizer implementation; inherits from RasterizerClass |

## Global / File-Static State
None.

## Key Methods

### SetView
- **Signature:** `virtual void SetView(view_data& View)`
- **Purpose:** Configure the camera/view parameters for OpenGL rendering
- **Inputs:** `View` (view_data reference containing camera/viewport configuration)
- **Outputs/Return:** None
- **Side effects:** Updates global OpenGL state via `OGL_SetView()`
- **Calls:** `OGL_SetView(View)`
- **Notes:** Must be called before any rendering. Virtual override of base class.

### SetForeground / SetForegroundView
- **Signature:** `virtual void SetForeground()` / `virtual void SetForegroundView(bool HorizReflect)`
- **Purpose:** Transition to foreground rendering mode for objects rendered over the main scene (weapons, UI layers)
- **Inputs:** `HorizReflect` (boolean flag for horizontal mirroring of foreground objects)
- **Outputs/Return:** None
- **Side effects:** Reconfigures OpenGL state for foreground layer
- **Calls:** `OGL_SetForeground()`, `OGL_SetForegroundView(HorizReflect)`

### Begin / End
- **Signature:** `void Begin()` / `void End()`
- **Purpose:** Bracket the main rendering pass; setup/cleanup OpenGL state
- **Outputs/Return:** None
- **Side effects:** Initializes and finalizes OpenGL rendering for a frame
- **Calls:** `OGL_StartMain()`, `OGL_EndMain()`

### texture_horizontal_polygon / texture_vertical_polygon
- **Signature:** `void texture_horizontal_polygon(polygon_definition& textured_polygon)` / similar
- **Purpose:** Render wall geometry with differing orientations (horizontal/vertical)
- **Inputs:** `textured_polygon` (polygon geometry and texture info)
- **Outputs/Return:** None
- **Side effects:** Issues OpenGL draw calls
- **Calls:** `OGL_RenderWall(textured_polygon, false/true)` (orientation flag)

### texture_rectangle
- **Signature:** `void texture_rectangle(rectangle_definition& textured_rectangle)`
- **Purpose:** Render sprite/rectangular geometry (floor/ceiling sprites, projectiles, decorations)
- **Inputs:** `textured_rectangle` (rectangle geometry and texture)
- **Outputs/Return:** None
- **Side effects:** Issues OpenGL draw calls
- **Calls:** `OGL_RenderSprite(textured_rectangle)`

## Control Flow Notes
Fits into the render pipeline as a frame-level orchestrator:
1. **Init:** Engine creates instance (implicit from class definition)
2. **Frame setup:** `SetView()` configures camera
3. **Main pass:** `Begin()` ΓåÆ render polygons/sprites ΓåÆ `End()`
4. **Foreground pass:** `SetForeground()` ΓåÆ `SetForegroundView()` ΓåÆ render weapon/UI geometry

All actual OpenGL commands are deferred to the `OGL_Render` module; this class acts purely as a virtual method dispatcher.

## External Dependencies
- **Includes:** `Rasterizer.h` (base class definition)
- **Imported functions (from OGL_Render module):** `OGL_SetView()`, `OGL_SetForeground()`, `OGL_SetForegroundView()`, `OGL_StartMain()`, `OGL_EndMain()`, `OGL_RenderWall()`, `OGL_RenderSprite()`
- **External types used:** `view_data`, `polygon_definition`, `rectangle_definition` (defined elsewhere in render headers)
- **Conditional compilation:** Only available if `HAVE_OPENGL` is defined; allows software or alternative rasterizers to coexist
