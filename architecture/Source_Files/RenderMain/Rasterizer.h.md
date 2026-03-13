# Source_Files/RenderMain/Rasterizer.h

## File Purpose
Abstract base class defining the interface for rasterizer implementations (software, OpenGL, etc.). Establishes the contract that all rasterizer subclasses must implement to handle view setup, polygon/rectangle rendering, and foreground object display.

## Core Responsibilities
- Provide virtual interface for setting view and rendering parameters
- Define entry/exit points for rendering passes (`Begin`, `End`)
- Specify rendering methods for textured polygons (horizontal and vertical)
- Specify rendering method for textured rectangles
- Support foreground object rendering (weapons, HUD elements in first-person)
- Manage horizontal reflection for foreground objects

## Key Types / Data Structures
None defined in this file.

## Global / File-Static State
None.

## Key Methods

### SetView
- Signature: `virtual void SetView(view_data& View)`
- Purpose: Configure rasterizer with current view parameters (position, orientation, FOV, screen dimensions)
- Inputs: Reference to `view_data` struct containing viewport configuration
- Outputs/Return: None
- Side effects: Updates rasterizer's internal view state
- Calls: None
- Notes: Must be called before any rendering operations; empty base implementation

### SetForeground
- Signature: `virtual void SetForeground()`
- Purpose: Switch rasterizer into foreground rendering mode for HUD elements (weapons in hand, etc.)
- Inputs: None
- Outputs/Return: None
- Side effects: Changes rendering mode context
- Calls: None
- Notes: Typically called after `End()` to render overlay objects

### SetForegroundView
- Signature: `virtual void SetForegroundView(bool HorizReflect)`
- Purpose: Configure view parameters for a foreground object with optional horizontal reflection
- Inputs: Boolean flag indicating whether foreground object is horizontally reflected
- Outputs/Return: None
- Side effects: Updates foreground rendering state
- Calls: None
- Notes: Used to mirror weapons/objects left-right in foreground display

### Begin / End
- Signature: `virtual void Begin()` / `virtual void End()`
- Purpose: Delimit a rendering pass; `Begin` prepares rasterizer, `End` finalizes output
- Inputs: None
- Outputs/Return: None
- Side effects: Manages rendering context lifecycle
- Calls: None
- Notes: Typical render frame: `Begin()` ΓåÆ polygon/rectangle methods ΓåÆ `End()`

### texture_horizontal_polygon
- Signature: `virtual void texture_horizontal_polygon(polygon_definition& textured_polygon)`
- Purpose: Render a textured polygon (walls, floors, ceilings) in horizontal orientation
- Inputs: Reference to polygon definition with texture and geometry data
- Outputs/Return: None
- Side effects: Draws geometry to framebuffer
- Calls: None
- Notes: Empty base implementation; subclasses implement actual rasterization

### texture_vertical_polygon
- Signature: `virtual void texture_vertical_polygon(polygon_definition& textured_polygon)`
- Purpose: Render a textured polygon in vertical orientation
- Inputs: Reference to polygon definition with texture and geometry data
- Outputs/Return: None
- Side effects: Draws geometry to framebuffer
- Calls: None
- Notes: Separate method from horizontal variant to support optimization by orientation

### texture_rectangle
- Signature: `virtual void texture_rectangle(rectangle_definition& textured_rectangle)`
- Purpose: Render a textured rectangular region (sprites, UI elements)
- Inputs: Reference to rectangle definition with texture and bounds
- Outputs/Return: None
- Side effects: Draws geometry to framebuffer
- Calls: None
- Notes: Used for non-polygonal geometry like sprites and HUD components

## Control Flow Notes
This class is instantiated as a concrete subclass and follows a typical render-loop pattern:
1. `SetView()` once per frame to configure viewport
2. `Begin()` to start main-view rendering
3. Series of `texture_*()` calls to render geometry
4. `SetForeground()` + `SetForegroundView()` to switch context
5. Additional rendering for foreground objects
6. `End()` to finalize frame

Subclasses (e.g., `OGL_Render.h`) implement these methods for specific rendering backends.

## External Dependencies
- **render.h** ΓÇö provides `view_data`, `polygon_definition`, `rectangle_definition` type definitions
- **OGL_Render.h** ΓÇö conditionally included (requires `HAVE_OPENGL`); OpenGL-specific declarations
- Standard C++ class and virtual method mechanism
