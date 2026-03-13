# Source_Files/RenderOther/HUDRenderer_OGL.h

## File Purpose
OpenGL-specific implementation of HUD rendering for the Aleph One game engine. Provides concrete methods for drawing HUD elements (motion sensor, shapes, text, UI rectangles) using OpenGL graphics calls.

## Core Responsibilities
- Update and render the motion sensor display with entity blips
- Draw shapes and textures at specified screen coordinates
- Render text with font and color support
- Fill and frame rectangles for UI elements
- Manage OpenGL clipping planes (e.g., for circular motion sensor viewport)
- Calculate text width for layout purposes
- Handle transparency and shape visibility in HUD rendering

## Key Types / Data Structures
None (inherits `HUD_Class` base definition and type system from HUDRenderer.h).

## Global / File-Static State
None.

## Key Functions / Methods

### update_motion_sensor
- Signature: `void update_motion_sensor(short time_elapsed)`
- Purpose: Update motion sensor state and blip positions each frame
- Inputs: Time elapsed since last update (ticks)
- Outputs/Return: None
- Side effects: Modifies internal motion sensor state
- Calls: (implementation in .cpp file)
- Notes: Virtual override of base class; called once per frame

### render_motion_sensor
- Signature: `void render_motion_sensor(short time_elapsed)`
- Purpose: Draw the motion sensor display and all entity blips to screen
- Inputs: Time elapsed (for animation/flashing effects)
- Outputs/Return: None
- Side effects: OpenGL rendering commands issued
- Calls: (implementation in .cpp file)
- Notes: Virtual override; typically called after update_motion_sensor

### DrawShape
- Signature: `void DrawShape(shape_descriptor shape, screen_rectangle *dest, screen_rectangle *src)`
- Purpose: Draw a sprite/shape with optional source/destination clipping
- Inputs: Shape ID, destination rect, optional source rect
- Outputs/Return: None
- Side effects: OpenGL texture binding and drawing commands
- Calls: (implementation in .cpp file)

### DrawText
- Signature: `void DrawText(const char *text, screen_rectangle *dest, short flags, short font_id, short text_color)`
- Purpose: Render text string to screen with specified font and color
- Inputs: Text string, destination rectangle, rendering flags, font ID, color index
- Outputs/Return: None
- Side effects: OpenGL text rasterization

### SetClipPlane / DisableClipPlane
- **SetClipPlane** signature: `void SetClipPlane(int x, int y, int c_x, int c_y, int radius)`
  - Purpose: Enable circular clipping plane (used for motion sensor circular viewport)
  - Inputs: Rectangle origin (x,y), circle center (c_x, c_y), radius
  - Side effects: OpenGL clipping plane state modified
- **DisableClipPlane** signature: `void DisableClipPlane(void)`
  - Purpose: Disable active clipping plane
  - Side effects: Restores full viewport rendering

### Remaining methods
- `draw_or_erase_unclipped_shape`, `draw_entity_blip`, `draw_message_area`, `DrawShapeAtXY`, `FillRect`, `FrameRect`, `DrawTexture`, `TextWidth`: Virtual overrides implementing base class interface; see HUDRenderer.h for signatures.

## Control Flow Notes
This class is instantiated as the active HUD renderer during gameplay. Each frame, the game engine calls `update_everything()` (from base class), which internally calls the motion sensor update/render pipeline and other HUD element drawing methods. All rendering uses OpenGL commands (likely immediate-mode or legacy OpenGL given the implementation style).

## External Dependencies
- `HUDRenderer.h` (base class `HUD_Class`, type definitions: `shape_descriptor`, `screen_rectangle`, `point2d`)
- OpenGL API (called in implementation .cpp)
- Types defined in bundled headers: `shape_descriptor`, `screen_rectangle`, `point2d`
