# Source_Files/RenderOther/HUDRenderer_OGL.cpp

## File Purpose
Implements OpenGL-based rendering for the HUD (Heads-Up Display), including dynamic UI elements, text, shapes, motion sensor, and interface graphics. Provides the primary HUD rendering pipeline integrating texture management, font rendering, and geometric primitives.

## Core Responsibilities
- Main HUD rendering orchestration (backdrop, dynamic elements, matrix setup)
- Motion sensor visualization with circular clipping
- Shape rendering with texture mapping and transparency control
- Text rendering with color and font specifications
- Primitive geometry (filled/framed rectangles)
- Texture loading and lifecycle management for HUD assets
- OpenGL state management (attributes, matrices, blending modes)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `HUD_OGL_Class` | class | HUD renderer implementation; inherits from `HUD_Class` and overrides virtual render methods |
| `OGL_Blitter` | class | Static HUD backdrop image storage and rendering |
| `TextureManager` | class | Manages shape texture setup, shading, transfer modes, and matrix state |
| `Shape_Blitter` | class | Renders shapes with scaling and rescaling support |
| `FontSpecifier` | class | Font properties and OpenGL text rendering |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `HUD_OGL` | `HUD_OGL_Class` | static | Singleton HUD renderer instance |
| `HUD_Blitter` | `OGL_Blitter` | static | Persistent HUD backdrop image texture |
| `hud_pict_not_found` | `bool` | static | Flag to prevent repeated failed backdrop load attempts |

## Key Functions / Methods

### OGL_DrawHUD
- **Signature:** `void OGL_DrawHUD(Rect &dest, short time_elapsed)`
- **Purpose:** Main entry point for rendering the entire HUD; loads backdrop, configures OpenGL state, applies scaling, marks dirty regions, and calls the update pipeline.
- **Inputs:** `dest` (destination rectangle in screen coords), `time_elapsed` (delta time in ticks)
- **Outputs/Return:** None (void)
- **Side effects:** Loads HUD backdrop texture on first call; sets OpenGL attributes (disables depth/alpha/blend/fog), manipulates modelview matrix, marks weapon/ammo/shield/oxygen/inventory displays as dirty, calls `HUD_OGL.update_everything()`
- **Calls:** `HUD_Blitter.Load()`, `HUD_Blitter.Draw()`, `OGL_RenderRect()`, `mark_*_display_as_dirty()` variants, `HUD_OGL.update_everything()`
- **Notes:** Caches backdrop load failure to avoid repeated attempts. Scales HUD coordinates from 640├ù160 logical space to actual destination rect. Uses `glPushAttrib/glPopMatrix` for state preservation.

### update_motion_sensor
- **Signature:** `void HUD_OGL_Class::update_motion_sensor(short time_elapsed)`
- **Purpose:** Conditionally updates motion sensor display if active and not disabled by game options.
- **Inputs:** `time_elapsed` (delta time in ticks)
- **Outputs/Return:** None
- **Side effects:** Calls `render_motion_sensor()` if conditions are met
- **Calls:** `render_motion_sensor()` (defined elsewhere)
- **Notes:** Checks `GET_GAME_OPTIONS()` and `MotionSensorActive` flag.

### DrawShape
- **Signature:** `void HUD_OGL_Class::DrawShape(shape_descriptor shape, screen_rectangle *dest, screen_rectangle *src)`
- **Purpose:** Renders a game shape texture into a destination rectangle with source-region clipping.
- **Inputs:** `shape` (descriptor for shape to render), `dest` (target screen rect), `src` (source rect within shape bitmap)
- **Outputs/Return:** None
- **Side effects:** Sets up `TextureManager`, enables `GL_TEXTURE_2D`, disables `GL_BLEND`, manipulates texture matrix, calls `OGL_RenderTexturedRect()`
- **Calls:** `get_shape_bitmap_and_shading_table()`, `TMgr.Setup()`, `TMgr.SetupTextureMatrix()`, `TMgr.RenderNormal()`, `OGL_RenderTexturedRect()`, `TMgr.RestoreTextureMatrix()`
- **Notes:** Calculates UV coordinates with scaling and offset based on source rect. Sets color to white (1, 1, 1) for full brightness.

### DrawShapeAtXY
- **Signature:** `void HUD_OGL_Class::DrawShapeAtXY(shape_descriptor shape, short x, short y, bool transparency = false)`
- **Purpose:** Renders a shape at absolute screen coordinates with optional alpha blending.
- **Inputs:** `shape`, `x`/`y` (screen coords), `transparency` (enable alpha blending)
- **Outputs/Return:** None
- **Side effects:** Conditionally enables `GL_BLEND` with alpha-to-one-minus-alpha blending if transparency is true
- **Calls:** (same setup/rendering pipeline as `DrawShape()`)
- **Notes:** Uses full shape dimensions (no source clipping). Enables `GL_BLEND` only if transparency requested.

### DrawTexture
- **Signature:** `void HUD_OGL_Class::DrawTexture(shape_descriptor texture, short texture_type, short x, short y, int size)`
- **Purpose:** Renders a shape as a centered, scaled texture within a bounding square.
- **Inputs:** `texture` (shape to render), `texture_type` (OGL texture category), `x`/`y` (top-left corner), `size` (square dimension)
- **Outputs/Return:** None
- **Side effects:** Creates `Shape_Blitter`, rescales based on aspect ratio, calls `b.OGL_Draw()`
- **Calls:** `Shape_Blitter` constructor, `Rescale()`, `OGL_Draw()`
- **Notes:** Maintains aspect ratio when rescaling; centers result within the bounding square. Early exit if blitter dimensions are zero.

### DrawText
- **Signature:** `void HUD_OGL_Class::DrawText(const char *text, screen_rectangle *dest, short flags, short font_id, short text_color)`
- **Purpose:** Renders colored text within a destination rectangle with specified font and alignment flags.
- **Inputs:** `text` (C-string), `dest` (bounding rect), `flags` (alignment/wrapping), `font_id` (font identifier), `text_color` (color table index)
- **Outputs/Return:** None
- **Side effects:** Looks up color and font from interface tables, sets OpenGL color, configures font filter
- **Calls:** `get_interface_color()`, `get_interface_font()`, `FontData.OGL_DrawText()`
- **Notes:** Color lookup is unsigned short (16-bit RGB). Font filter inherited from HUD texture configuration.

### FillRect
- **Signature:** `void HUD_OGL_Class::FillRect(screen_rectangle *r, short color_index)`
- **Purpose:** Renders a filled rectangle in a specified interface color.
- **Inputs:** `r` (rectangle bounds), `color_index` (color table index)
- **Outputs/Return:** None
- **Side effects:** Sets OpenGL color, calls `OGL_RenderRect()`
- **Calls:** `get_interface_color()`, `OGL_RenderRect()`

### FrameRect
- **Signature:** `void HUD_OGL_Class::FrameRect(screen_rectangle *r, short color_index)`
- **Purpose:** Renders a rectangle outline (border) in a specified interface color.
- **Inputs:** `r`, `color_index` (same as `FillRect`)
- **Outputs/Return:** None
- **Side effects:** Calls `OGL_RenderFrame()` with 1-pixel border, expanded by 1 pixel on all sides
- **Calls:** `get_interface_color()`, `OGL_RenderFrame()`

### SetClipPlane
- **Signature:** `void HUD_OGL_Class::SetClipPlane(int x, int y, int c_x, int c_y, int radius)`
- **Purpose:** Configures an OpenGL clip plane tangent to a circle, used to clip motion sensor blips to circular boundary.
- **Inputs:** `x`/`y` (blip coordinates), `c_x`/`c_y` (circle center), `radius` (circle radius)
- **Outputs/Return:** None
- **Side effects:** Enables `GL_CLIP_PLANE0`, calls `glClipPlane()`
- **Calls:** `glEnable()`, `glClipPlane()`
- **Notes:** Computes tangent point by normalizing blip vector and scaling by radius. Returns early if blip distance Γëñ 2.0 (too close to center).

### DisableClipPlane
- **Signature:** `void HUD_OGL_Class::DisableClipPlane(void)`
- **Purpose:** Disables OpenGL clip plane.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `glDisable(GL_CLIP_PLANE0)`
- **Calls:** `glDisable()`

### TextWidth
- **Signature:** `int HUD_OGL_Class::TextWidth(const char* text, short font_id)`
- **Purpose:** Queries text width in pixels for a given font.
- **Inputs:** `text`, `font_id`
- **Outputs/Return:** Text width in pixels
- **Calls:** `get_interface_font()`, `FontSpecifier::TextWidth()`

### draw_message_area
- **Signature:** `void HUD_OGL_Class::draw_message_area(short)`
- **Purpose:** Renders the player name display area with network panel backdrop.
- **Inputs:** Unused short parameter
- **Outputs/Return:** None
- **Side effects:** Draws network panel shape at offset position, calls `draw_player_name()`
- **Calls:** `get_interface_rectangle()`, `DrawShapeAtXY()`, `draw_player_name()`
- **Notes:** Applies hardcoded offset (`MESSAGE_AREA_X_OFFSET`, `MESSAGE_AREA_Y_OFFSET`) to position.

## Control Flow Notes
`OGL_DrawHUD()` is the frame-time entry point, called during the HUD rendering phase after world geometry is rendered. It orchestrates:
1. Lazy-load HUD backdrop texture
2. Push OpenGL attribute state; clear rendering flags (depth, alpha, blend, fog)
3. Set up modelview matrix for HUD coordinate system (640├ù160 logical space scaled to dest rect)
4. Render static backdrop or fallback solid rectangle
5. Mark all dynamic displays as dirty
6. Call `HUD_OGL.update_everything()` (defined in base class) to render dynamic content
7. Restore matrix and attributes

Motion sensor and shape rendering operate during the `update_everything()` call, which invokes inherited HUD methods that use `SetClipPlane()` for circular clipping.

## External Dependencies
- **OpenGL API:** `glPushAttrib`, `glPopAttrib`, `glDisable`, `glEnable`, `glMatrixMode`, `glPushMatrix`, `glPopMatrix`, `glLoadIdentity`, `glTranslated`, `glScaled`, `glColor3*`, `glClipPlane`
- **Rendering modules:** `OGL_RenderRect()`, `OGL_RenderFrame()`, `OGL_RenderTexturedRect()` (defined elsewhere)
- **Texture management:** `OGL_Blitter`, `TextureManager`, `Shape_Blitter`, `TxtrTypeInfoList[OGL_Txtr_HUD]`
- **Font system:** `FontHandler.h`, `get_interface_font()`, `FontSpecifier`
- **Game state:** `MotionSensorActive`, `GET_GAME_OPTIONS()`, `current_player_index`, game option flags
- **Interface resources:** `get_interface_color()`, `get_interface_rectangle()`, shape descriptors, interface collections
- **Utility:** `get_shape_bitmap_and_shading_table()`, `LuaTexturePaletteSize()` (Lua integration), `BUILD_DESCRIPTOR()` macros
