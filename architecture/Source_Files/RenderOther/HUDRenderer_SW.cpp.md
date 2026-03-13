# Source_Files/RenderOther/HUDRenderer_SW.cpp

## File Purpose
Software-rasterized HUD rendering implementation for the Aleph One game engine. Provides concrete implementations of the `HUD_Class` interface using SDL surfaces, enabling 2D UI composition via shape blitting, text rendering, and primitive drawing operations.

## Core Responsibilities
- Update and render the motion sensor indicator when game state changes
- Draw 2D shapes from the shapes resource file via delegation to screen drawing functions
- Draw scaled textured elements (e.g., weapon textures, HUD decorations) using `Shape_Blitter`
- Render text strings with color and font selection
- Draw filled and outlined rectangles for UI primitives
- Rotate SDL surfaces 90┬░ for orientation transformations
- Serve as the software rendering backend, complementary to an OpenGL path

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `HUD_SW_Class` | class | Software-rasterized HUD renderer; inherits from `HUD_Class` |
| `screen_rectangle` | struct | Screen region bounds for drawing |
| `shape_descriptor` | typedef | Opaque identifier for a shape resource (collection + index) |
| `SDL_Surface` | struct | SDL graphics surface (pixels + format metadata) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `HUD_Buffer` | `SDL_Surface*` | extern | Target surface for HUD composition; provided by caller |
| `MotionSensorActive` | `bool` | extern | Runtime flag indicating if motion sensor should render |

## Key Functions / Methods

### update_motion_sensor
- **Signature:** `void update_motion_sensor(short time_elapsed)`
- **Purpose:** Conditionally update and redraw the motion sensor display if the game state changed or a full redraw is forced.
- **Inputs:** `time_elapsed` ΓÇö milliseconds since last update (or `NONE` to force full refresh)
- **Outputs/Return:** None; side effects only
- **Side effects:** Calls `render_motion_sensor()`, sets `ForceUpdate` flag, draws motion sensor mount shape via `DrawShapeAtXY()`
- **Calls:** `GET_GAME_OPTIONS()`, `motion_sensor_has_changed()`, `render_motion_sensor()`, `get_interface_rectangle()`, `DrawShapeAtXY()`, `BUILD_DESCRIPTOR()`
- **Notes:** Guarded by game options check (`_motion_sensor_does_not_work`) and `MotionSensorActive` flag; `time_elapsed == NONE` triggers unconditional redraw

### DrawTexture
- **Signature:** `void DrawTexture(shape_descriptor shape, short texture_type, short x, short y, int size)`
- **Purpose:** Draw a textured UI element (e.g., weapon sprite, map icon) with automatic aspect-ratio-preserving scaling into a square bounding box.
- **Inputs:** `shape` ΓÇö shape resource; `texture_type` ΓÇö classification (wall/sprite/weapon/interface); `x`, `y` ΓÇö top-left destination; `size` ΓÇö bounding square size in pixels
- **Outputs/Return:** None; draws to `HUD_Buffer`
- **Side effects:** Creates `Shape_Blitter`, rescales, renders to `HUD_Buffer`
- **Calls:** `Shape_Blitter::Rescale()`, `Shape_Blitter::Width()`, `Shape_Blitter::Height()`, `Shape_Blitter::SDL_Draw()`
- **Notes:** Preserves aspect ratio; centers scaled image within bounding box; skips drawing if width or height is zero

### rotate_surface
- **Signature:** `SDL_Surface* rotate_surface(SDL_Surface *s, int width, int height)`
- **Purpose:** Rotate an SDL surface 90┬░ clockwise, handling multiple pixel depths (8/16/32-bit).
- **Inputs:** `s` ΓÇö source surface; `width`, `height` ΓÇö source dimensions
- **Outputs/Return:** New allocated `SDL_Surface` with swapped dimensions; returns null if input is null
- **Side effects:** Allocates new SDL surface; copies palette if present
- **Calls:** `SDL_CreateRGBSurface()`, `rotate<T>()` (template), `SDL_SetPaletteColors()`
- **Notes:** Dispatcher function that selects pixel-size-specific rotation; caller responsible for freeing returned surface

### rotate (template)
- **Signature:** `template <class T> static void rotate(T *src_pixels, int src_pitch, T *dst_pixels, int dst_pitch, int width, int height)`
- **Purpose:** In-place pixel-by-pixel rotation of typed pixel data; transpose operation mapping `[y][x]` ΓåÆ `[x][y]`.
- **Inputs:** Source and destination pixel arrays (pitch in units, not bytes); dimensions
- **Outputs/Return:** None; writes to `dst_pixels`
- **Side effects:** Modifies destination buffer
- **Calls:** None
- **Notes:** Generic over pixel type (`pixel8`, `pixel16`, `pixel32`); pitch parameter accounts for type size

## Trivial Delegation Helpers
The following methods are thin wrappers delegating to external drawing functions:
- `DrawShape(ΓÇª)` ΓåÆ `_draw_screen_shape()`
- `DrawShapeAtXY(ΓÇª)` ΓåÆ `_draw_screen_shape_at_x_y()`
- `DrawText(ΓÇª)` ΓåÆ `_draw_screen_text()`
- `TextWidth(ΓÇª)` ΓåÆ `_text_width()`
- `FillRect(ΓÇª)` ΓåÆ `_fill_rect()`
- `FrameRect(ΓÇª)` ΓåÆ `_frame_rect()`

## Control Flow Notes
This class is instantiated as the HUD renderer and called during the game's render phase (once per frame). The `update_motion_sensor()` method is invoked periodically to check for motion sensor state changes and redraw if needed. All drawing methods are called by the parent `HUD_Class` framework to composite UI elements onto `HUD_Buffer`, which is later presented to the display. The software path is a fallback/alternative to OpenGL rendering.

## External Dependencies
- **Includes:** `HUDRenderer_SW.h` (class definition), `images.h` (image resource access), `shell.h` (shape surface retrieval), `Shape_Blitter.h` (textured shape rendering)
- **External functions:** `_draw_screen_shape()`, `_draw_screen_shape_at_x_y()`, `_draw_screen_text()`, `_text_width()`, `_fill_rect()`, `_frame_rect()`, `motion_sensor_has_changed()`, `render_motion_sensor()`, `get_interface_rectangle()`, `GET_GAME_OPTIONS()`, `BUILD_DESCRIPTOR()`, `GET_COLLECTION()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_DESCRIPTOR_SHAPE()`, `GET_COLLECTION_CLUT()`
- **SDL symbols:** `SDL_CreateRGBSurface()`, `SDL_SetPaletteColors()`, `SDL_Rect`, `SDL_Surface`, `SDL_FreeSurface()`
- **External state:** `HUD_Buffer` (target drawing surface), `MotionSensorActive` (motion sensor enabled flag)
