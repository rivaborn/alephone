# Source_Files/RenderOther/HUDRenderer_SW.h

## File Purpose
Software (CPU/memory-based) implementation of HUD rendering for Aleph One. Provides concrete graphics operations using screen drawing primitivesΓÇöan alternative to hardware/GPU-accelerated HUD rendering. Written in 2001 as part of the Bungie/Aleph One codebase.

## Core Responsibilities
- Override pure virtual base methods from `HUD_Class` with software rendering implementations
- Update and render motion sensor (radar) display and entity blips
- Draw shapes, text, and rectangles at screen coordinates
- Manage texture rendering and clipping regions
- Calculate text dimensions for layout purposes
- Handle shape drawing with optional transparency and source/dest clipping

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `HUD_SW_Class` | class | Software HUD renderer; concrete implementation of `HUD_Class` pure virtual interface |

## Global / File-Static State
None.

## Key Functions / Methods

### update_motion_sensor
- **Signature:** `void update_motion_sensor(short time_elapsed)` (protected, override)
- **Purpose:** Update internal state of motion sensor display (e.g., radar blip positions, animation)
- **Inputs:** `time_elapsed` ΓÇô milliseconds since last update
- **Outputs/Return:** None (modifies internal state)
- **Side effects:** Updates motion sensor state; called from base class `update_everything()`

### render_motion_sensor
- **Signature:** `void render_motion_sensor(short time_elapsed)` (protected, override)
- **Purpose:** Draw motion sensor visuals to screen
- **Inputs:** `time_elapsed` ΓÇô for animation timing
- **Outputs/Return:** None (screen drawing side effect)
- **Side effects:** Draws to framebuffer

### draw_or_erase_unclipped_shape
- **Signature:** `void draw_or_erase_unclipped_shape(short x, short y, shape_descriptor shape, bool draw)` (protected, override)
- **Purpose:** Draw or erase a shape without applying screen clipping planes
- **Inputs:** `(x, y)` ΓÇô screen coordinates; `shape` ΓÇô shape ID; `draw` ΓÇô true to draw, false to erase
- **Outputs/Return:** None
- **Side effects:** Framebuffer modification

### DrawShape
- **Signature:** `void DrawShape(shape_descriptor shape, screen_rectangle *dest, screen_rectangle *src)` (protected, override)
- **Purpose:** Draw shape with source and destination clipping rectangles
- **Inputs:** `shape` ΓÇô shape identifier; `dest` ΓÇô destination screen rect; `src` ΓÇô source texture rect (for clipping source art)
- **Outputs/Return:** None
- **Side effects:** Framebuffer write

### DrawShapeAtXY
- **Signature:** `void DrawShapeAtXY(shape_descriptor shape, short x, short y, bool transparency = false)` (protected, override)
- **Purpose:** Convenience: draw shape at absolute screen position
- **Inputs:** `(x, y)` ΓÇô top-left corner; `transparency` ΓÇô enable alpha blending
- **Outputs/Return:** None
- **Side effects:** Framebuffer write

### DrawText
- **Signature:** `void DrawText(const char *text, screen_rectangle *dest, short flags, short font_id, short text_color)` (protected, override)
- **Purpose:** Render text string with formatting
- **Inputs:** `text` ΓÇô null-terminated string; `dest` ΓÇô bounding rectangle; `flags` ΓÇô text alignment/style; `font_id` ΓÇô font selection; `text_color` ΓÇô color index
- **Outputs/Return:** None
- **Side effects:** Framebuffer write

### FillRect / FrameRect
- **Signature:** `void FillRect(screen_rectangle *r, short color_index)` / `void FrameRect(screen_rectangle *r, short color_index)` (protected, override)
- **Purpose:** Fill or outline a rectangle with a palette color
- **Inputs:** `r` ΓÇô screen rectangle; `color_index` ΓÇô palette index
- **Outputs/Return:** None
- **Side effects:** Framebuffer write

### DrawTexture
- **Signature:** `void DrawTexture(shape_descriptor shape, short texture_type, short x, short y, int size)` (protected, override)
- **Purpose:** Draw a textured/scaled element (e.g., weapon panel, ammo icons)
- **Inputs:** `shape` ΓÇô shape ID; `texture_type` ΓÇô rendering mode; `(x, y)` ΓÇô position; `size` ΓÇô scale
- **Outputs/Return:** None
- **Side effects:** Framebuffer write

### SetClipPlane / DisableClipPlane
- **Signature:** `void SetClipPlane(int x, int y, int c_x, int c_y, int radius)` / `void DisableClipPlane(void)` (protected, override)
- **Purpose:** (Stub) clipping region management
- **Outputs/Return:** None (no-op implementations)
- **Notes:** Empty implementations; clipping likely handled per-call instead of as global state

### TextWidth
- **Signature:** `int TextWidth(const char* text, short font_id)` (public override)
- **Purpose:** Calculate on-screen pixel width of a text string for layout
- **Inputs:** `text` ΓÇô string; `font_id` ΓÇô font
- **Outputs/Return:** Pixel width
- **Notes:** Used by base class for text positioning; only public method

## Control Flow Notes
- Part of the HUD rendering subsystem invoked per-frame by `update_everything()` (from base class)
- `update_motion_sensor()` ΓåÆ `render_motion_sensor()` pipeline for radar display
- Other methods (`DrawShape`, `DrawText`, `FillRect`) called by base class logic to render UI panels, inventory, weapon status, and overlays
- All drawing is immediate-mode (direct to framebuffer); no deferred rendering

## External Dependencies
- **Includes:** `HUDRenderer.h` (base class, constants, data structures)
- **Defined elsewhere:** `shape_descriptor`, `screen_rectangle`, `point2d` types; shape/texture/font/color constants; `SoundManager`, `motion_sensor`, player/weapon/item subsystems (included transitively via HUDRenderer.h)
