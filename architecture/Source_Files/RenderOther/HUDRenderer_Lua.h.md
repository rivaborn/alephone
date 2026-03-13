# Source_Files/RenderOther/HUDRenderer_Lua.h

## File Purpose
Implements a Lua-compatible HUD renderer class that extends the base `HUD_Class`. Provides drawing primitives and motion sensor blip management for scripted HUD themes, bridging Lua script logic with native rendering operations.

## Core Responsibilities
- Manage entity blips (radar markers) for motion sensor display
- Provide drawing primitives (filled/framed rectangles, text, images, shapes)
- Handle masking modes for clipped rendering regions
- Update and render motion sensor display state per frame
- Maintain drawing context (start/end draw, apply clipping)
- Track drawing/masking state across render lifecycle

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `blip_info` | struct | Stores metadata about a single radar blip: object type, intensity, distance, direction |
| `HUD_Lua_Class` | class | Main renderer class inheriting from `HUD_Class`; provides Lua-facing HUD rendering API |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_blips` | `std::vector<blip_info>` | member | Dynamic list of active radar blips |
| `m_drawing` | `bool` | member | Tracks whether currently in drawing state |
| `m_opengl` | `bool` | member | Indicates if using OpenGL rendering path |
| `m_surface` | `SDL_Surface*` | member | SDL surface for software rendering or mask buffer |
| `m_wr` | `SDL_Rect` | member | Window/render rectangle bounds |
| `m_masking_mode` | `short` | member | Current masking mode (disabled/enabled/drawing/erasing) |

## Key Functions / Methods

### update_motion_sensor
- Signature: `void update_motion_sensor(short time_elapsed)`
- Purpose: Update motion sensor display state based on elapsed time
- Inputs: `time_elapsed` ΓÇô milliseconds since last frame
- Outputs/Return: (none)
- Side effects: Updates internal motion sensor state
- Calls: (derived from virtual override; actual calls in subclass)

### clear_entity_blips / add_entity_blip
- Signature: `void clear_entity_blips(void)` / `void add_entity_blip(short mtype, short intensity, short x, short y)`
- Purpose: Manage radar blip collection (clear all or add new blip)
- Inputs: `mtype` ΓÇô object type, `intensity` ΓÇô blip strength, `x`/`y` ΓÇô screen coordinates
- Outputs/Return: (none)
- Side effects: Modifies `m_blips` vector

### entity_blip_count / entity_blip
- Signature: `size_t entity_blip_count(void)` / `blip_info entity_blip(size_t index)`
- Purpose: Query blip collection (count or retrieve by index)
- Outputs/Return: Count or `blip_info` struct
- Notes: Likely bounds-unchecked; caller responsible for valid indices

### start_draw / end_draw / apply_clip
- Signature: `void start_draw(void)`, `void end_draw(void)`, `void apply_clip(void)`
- Purpose: Bracket drawing operations and apply clipping region
- Side effects: Modifies OpenGL/SDL rendering state

### masking_mode / set_masking_mode / clear_mask
- Signature: `short masking_mode(void)`, `void set_masking_mode(short masking_mode)`, `void clear_mask(void)`
- Purpose: Query/set masking mode (for stencil or clipping operations) or clear mask buffer
- Notes: Modes are enum `_mask_disabled`, `_mask_enabled`, `_mask_drawing`, `_mask_erasing`

### fill_rect / frame_rect
- Signature: `void fill_rect(float x, float y, float w, float h, float r, float g, float b, float a)` / `void frame_rect(..., float t)`
- Purpose: Draw filled or outlined rectangle with RGBA color
- Inputs: Position, dimensions (floats), RGBA channels, stroke thickness (frame only)
- Side effects: Modifies rendering target (surface or backbuffer)

### draw_text / draw_image / draw_shape
- Signature: `void draw_text(FontSpecifier *font, const char *text, float x, float y, float r, float g, float b, float a, float scale)` / similar for image/shape
- Purpose: Render text, image, or shape at given coordinates with color/scale
- Inputs: Pointer to font/image/shape object, text string (text only), position, RGBA, scale (text only)
- Side effects: GPU/surface drawing operations

### start_using_mask / end_using_mask / start_drawing_mask / end_drawing_mask
- Signature: Protected helper methods
- Purpose: Manage stencil/clipping mask lifecycle for masked rendering regions
- Inputs: `erase` ΓÇô whether to erase (vs. draw) mask in drawing phase
- Side effects: Enable/disable stencil test, manipulate stencil buffer

### render_motion_sensor / draw_all_entity_blips
- Signature: Protected virtual overrides
- Purpose: Render motion sensor display and draw all collected radar blips
- Side effects: GPU drawing operations

## Control Flow Notes
- **Frame-based**: `update_motion_sensor()` called once per frame with elapsed time to advance animations/decay
- **Drawing phase**: Bracketed by `start_draw()`/`end_draw()` around primitive calls
- **Masking support**: Lua scripts can enable masking mode to draw clipped regions (e.g., radar radial clips)
- **Initialization**: Likely instantiated once; `Lua_HUDInstance()` returns singleton or current instance
- **Global entry point**: `Lua_DrawHUD(short time_elapsed)` called externally to trigger frame render

## External Dependencies
- **Base class**: `HUD_Class` (HUDRenderer.h) ΓÇö provides `update_everything()` and virtual method framework
- **SDL**: `SDL_Surface`, `SDL_Rect` ΓÇö software rendering surfaces
- **Forward declarations**: `FontSpecifier`, `Image_Blitter`, `Shape_Blitter` ΓÇö game-specific rendering abstraction classes
- **Game types** (defined elsewhere): `world_distance`, `angle`, `shape_descriptor`, `screen_rectangle`, `point2d`
- **Exceptions**: Throws `std::logic_error` from `TextWidth()` override to indicate Lua path does not use it
