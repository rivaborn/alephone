# Source_Files/RenderOther/HUDRenderer_Lua.cpp

## File Purpose
Implements the Lua HUD renderer, providing C++ bindings and dual-path (OpenGL/SDL) rendering for Lua-scripted HUD themes. Manages motion sensor blips, drawing primitives (rectangles, text, images, shapes), and stencil-based masking modes for HUD composition.

## Core Responsibilities
- **Blip management**: track entity radar markers with position, intensity, and type
- **Rendering state**: initialize/finalize OpenGL or SDL surfaces for HUD frame
- **Masking modes**: manage stencil test state for compositing (disabled/enabled/drawing/erasing)
- **Clipping**: apply scissor test (OpenGL) or clip rects (SDL) to drawing operations
- **Drawing primitives**: filled/framed rectangles, text, pre-rendered images, game shapes
- **Dual rendering paths**: transparent abstraction over OpenGL (shader-less, immediate mode) and SDL software rendering
- **Coordinate transform**: translate HUD coordinates to screen space; matrix management for OpenGL

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `blip_info` | struct | Radar marker: monster type, intensity, distance, angle |
| `HUD_Lua_Class` | class | Main renderer; inherits `HUD_Class`; holds blips, surfaces, state |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `HUD_Lua` | `HUD_Lua_Class` | static | Singleton HUD renderer instance |

## Key Functions / Methods

### Lua_HUDInstance
- Signature: `HUD_Lua_Class *Lua_HUDInstance()`
- Purpose: Accessor for the singleton HUD renderer
- Outputs/Return: Pointer to static `HUD_Lua`
- Notes: Called from Lua script layer to obtain renderer handle

### Lua_DrawHUD
- Signature: `void Lua_DrawHUD(short time_elapsed)`
- Purpose: Frame-level entry point; orchestrates HUD update and render
- Inputs: Elapsed milliseconds since last frame
- Side effects: Calls `L_Call_HUDDraw()` (Lua script callback)
- Calls: `update_motion_sensor()`, `start_draw()`, `L_Call_HUDDraw()`, `end_draw()`
- Notes: Invoked once per frame from game loop

### add_entity_blip
- Signature: `void HUD_Lua_Class::add_entity_blip(short mtype, short intensity, short x, short y)`
- Purpose: Register a radar blip at world coordinates
- Inputs: Monster type, intensity, x/y world coords (relative to player origin)
- Side effects: Appends to `m_blips` vector
- Calls: `distance2d()`, `arctangent()` (geometry helpers, defined elsewhere)
- Notes: Blips accumulate; caller must `clear_entity_blips()` each frame

### start_draw
- Signature: `void HUD_Lua_Class::start_draw(void)`
- Purpose: Initialize rendering context (OpenGL or SDL) for HUD frame
- Side effects: Binds screen, sets GL state (matrix, blend, depth/stencil disable), or creates/clears SDL surface; sets `m_drawing=true`
- Calls: `Screen::instance()`, `get_screen_mode()`, `MainScreenSurface()` (defined elsewhere)
- Notes: Must pair with `end_draw()`. OpenGL: saves attrib bits, identity matrix, translates to window rect. SDL: allocates/reuses BGRA8888 surface or reuses existing

### end_draw
- Signature: `void HUD_Lua_Class::end_draw(void)`
- Purpose: Restore rendering context after HUD frame
- Side effects: Sets `m_drawing=false`; pops OpenGL matrix and attrib bits if GL mode
- Notes: Inverse of `start_draw()`; noop if SDL

### apply_clip
- Signature: `void HUD_Lua_Class::apply_clip(void)`
- Purpose: Apply clipping rectangle from `Screen::lua_clip_rect` to current render target
- Side effects: Enables scissor test (GL) or sets clip rect (SDL); clamps to window bounds
- Calls: `Screen::instance()`, `scissor_screen_to_rect()`, `SDL_SetClipRect()`
- Notes: Called by all drawing methods; accounting for window offset

### set_masking_mode
- Signature: `void HUD_Lua_Class::set_masking_mode(short masking_mode)`
- Purpose: Transition masking mode (disabled ΓåÆ enabled ΓåÆ drawing ΓåÆ erasing, etc.)
- Inputs: Target mode (enum: `_mask_disabled`, `_mask_enabled`, `_mask_drawing`, `_mask_erasing`)
- Side effects: End previous mode, set new mode, start new mode; modifies GL stencil/alpha test state
- Calls: `end_drawing_mask()`, `end_using_mask()`, `start_drawing_mask()`, `start_using_mask()`
- Notes: Validates mode range; noop if already in target mode

### fill_rect / frame_rect / draw_text / draw_image / draw_shape
- **fill_rect**: Renders solid RGBA rectangle; OpenGL uses `OGL_RenderRect()`, SDL uses `SDL_FillRect()` + blit
- **frame_rect**: Draws rectangle outline (thickness `t`); decomposed into 4 rects (top, bottom, left, right borders) for SDL
- **draw_text**: Renders text via `FontSpecifier`; OpenGL uses `OGL_Render()` with matrix stack, SDL uses `font->Info->draw_text()`; scaling supported in GL (native), approximated in SDL (disabled by FIXME)
- **draw_image**: Delegates to `Image_Blitter::Draw()` with adjusted coordinates
- **draw_shape**: Delegates to `Shape_Blitter::OGL_Draw()` or `SDL_Draw()` depending on renderer
- All call `apply_clip()` and return early if `!m_drawing` or invalid dimensions

## Control Flow Notes
- **Init**: `Lua_DrawHUD()` ΓåÆ `start_draw()` (per-frame setup)
- **Update**: `update_motion_sensor()` (game state)
- **Render**: Lua script calls `fill_rect()`, `draw_text()`, etc. via C bindings
- **Cleanup**: `end_draw()` (per-frame teardown)
- Masking modes allow Lua to composite images using stencil buffer (e.g., radial vignette, masked HUD sections)

## External Dependencies
- **FontHandler.h**: `FontSpecifier` class for text rendering
- **Image_Blitter.h**, **Shape_Blitter.h**: Sprite/bitmap rendering abstractions
- **lua_hud_script.h**: Lua callback entry point `L_Call_HUDDraw()`
- **shell.h**: `get_screen_mode()`, `MainScreenSurface()`, etc.
- **screen.h**: `alephone::Screen` singleton, viewport/clip management
- **images.h**: Resource loading (used elsewhere, not directly here)
- **OGL_Headers.h / OGL_Render.h**: OpenGL state, `OGL_RenderRect()`, `OGL_RenderFrame()`
- **SDL2/SDL.h**: Surface creation, blitting, pixel format conversion
- **math.h**: `ceilf()` (disabled scaling code)
- External symbols: `MotionSensorActive` (bool, state flag); `distance2d()`, `arctangent()` (geometry, defined elsewhere); `L_Call_HUDDraw()` (Lua)
