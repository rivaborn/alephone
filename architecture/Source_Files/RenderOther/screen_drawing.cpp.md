# Source_Files/RenderOther/screen_drawing.cpp

## File Purpose
Low-level SDL2-based rendering implementation for the Aleph One game engine's UI and HUD. Handles drawing shapes, text (bitmap and TrueType), rectangles, lines, and polygons to SDL surfaces with clipping support. Manages interface colors, fonts, layout rectangles, and provides an abstraction layer for switching drawing targets.

## Core Responsibilities
- **Port abstraction**: Redirect drawing operations to different SDL surfaces (main screen, HUD buffer, terminal, intro, map, custom)
- **Shape rendering**: Blit sprite graphics with optional source clipping
- **Text rendering**: Render both SDL bitmap fonts and TrueType fonts with styling (bold, italic, underline), alignment, and word wrapping
- **Geometric primitives**: Fill/frame rectangles, draw thin/thick lines with clipping, fill convex polygons with Sutherland-Hodgman clipping
- **Interface resource management**: Store and vend hardcoded interface rectangles, colors, and font specifications
- **Clip rectangle management**: Enable/disable drawing clipping regions for all drawing operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| screen_rectangle | struct | Portable rectangle (top, left, bottom, right); used for interface layout |
| rgb_color | struct | 16-bit RGB color (red, green, blue fields) |
| FontSpecifier | class | Font specification with name, size, style; used for interface text |
| sdl_font_info | struct | SDL bitmap font with character metrics; defined elsewhere |
| ttf_font_info | struct | TrueType font wrapper; defined elsewhere |
| span_t | struct (static, local) | Horizontal span for polygon scan-fill (left/right bounds per scanline) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| interface_rectangles | screen_rectangle[32] | static | Hardcoded layout rectangles for all UI elements |
| InterfaceFonts | FontSpecifier[7] | static | Initialized font specifications for interface, weapons, terminal, etc. |
| InterfaceColors | rgb_color[26] | static | Predefined color palette for interface and player colors |
| draw_surface | SDL_Surface* | global | Current rendering target (main screen or off-screen buffer) |
| old_draw_surface | SDL_Surface* | static | Saved previous surface for _restore_port() |
| draw_clip_rect_active | bool | global | Flag: clipping rectangle is active |
| draw_clip_rect | screen_rectangle | global | Current clipping bounds |

## Key Functions / Methods

### initialize_screen_drawing
- Signature: `void initialize_screen_drawing(void)`
- Purpose: One-time initialization of interface system; loads rectangles, colors, and initializes font objects
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes interface_rectangles, InterfaceColors, InterfaceFonts globals; calls FontSpecifier::Init()
- Calls: load_interface_rectangles(), load_screen_interface_colors(), InterfaceFonts[loop].Init()
- Notes: Calls to rectangle/color loaders are currently empty stubs (XML-configurable in future)

### _set_port_to_* family (_set_port_to_screen_window, _set_port_to_gworld, _set_port_to_HUD, _set_port_to_term, _set_port_to_intro, _set_port_to_map, _set_port_to_custom)
- Signature: `void _set_port_to_screen_window(void)` and 6 variants; _set_port_to_custom takes SDL_Surface*
- Purpose: Redirect all subsequent drawing operations to a different SDL surface
- Inputs: _set_port_to_custom takes target surface pointer
- Outputs/Return: None
- Side effects: Updates draw_surface and old_draw_surface; sets intro_buffer_changed flag if intro
- Calls: None (state mutation only)
- Notes: Asserts old_draw_surface==NULL to prevent nesting; _restore_port() undoes this

### _draw_screen_shape
- Signature: `void _draw_screen_shape(shape_descriptor shape_id, screen_rectangle *destination, screen_rectangle *source)`
- Purpose: Render a sprite/shape graphic to current surface with destination and optional source clipping
- Inputs: shape_id (descriptor), destination rect, source rect (NULL = use full shape)
- Outputs/Return: None
- Side effects: Blits to draw_surface; updates main screen dirty rect if draw_surface==MainScreenSurface()
- Calls: get_shape_surface(), SDL_BlitSurface(), MainScreenUpdateRects(), SDL_FreeSurface()
- Notes: Converts screen_rectangle to SDL_Rect; gracefully returns if get_shape_surface returns NULL; frees shape surface after use

### _draw_screen_shape_at_x_y
- Signature: `void _draw_screen_shape_at_x_y(shape_descriptor shape_id, short x, short y)`
- Purpose: Draw shape at absolute pixel coordinates using shape's full dimensions
- Inputs: shape_id, x, y position
- Outputs/Return: None
- Side effects: Blits to draw_surface; updates main screen if needed
- Calls: get_shape_surface(), SDL_BlitSurface(), MainScreenUpdateRects(), SDL_FreeSurface()
- Notes: Simplified version of _draw_screen_shape without source clipping

### draw_glyph (template function)
- Signature: `template<class T> inline static int draw_glyph(uint8 c, int x, int y, T *p, int pitch, int clip_left, int clip_top, int clip_right, int clip_bottom, uint32 pixel, const sdl_font_info *font, bool oblique)`
- Purpose: Render single character glyph directly to pixel buffer with per-pixel clipping
- Inputs: character code, x/y position, pixel buffer pointer, pitch (bytes per scanline), clipping bounds, color, font, italic flag
- Outputs/Return: Glyph advance width (spacing to next character)
- Side effects: Writes directly to pixel buffer; modifies pointer/scanline addresses during clipping
- Calls: None (inline pixel access)
- Notes: Applies ascent/kerning from font; clips on all 4 sides; supports oblique (italic) by offset; returns advance even if clipped off-screen

### draw_text (template function)
- Signature: `template<class T> inline static int draw_text(const uint8 *text, size_t length, int x, int y, T *p, int pitch, int clip_left, int clip_top, int clip_right, int clip_bottom, uint32 pixel, const sdl_font_info *font, uint16 style)`
- Purpose: Render text string to pixel buffer with styling (bold, italic, underline)
- Inputs: text, length, position, buffer, pitch, clipping, color, font, style flags
- Outputs/Return: Total width of rendered text
- Side effects: Calls draw_glyph for each character; modifies pixel buffer
- Calls: draw_glyph() per character in loop
- Notes: Skips characters outside font range; applies styleBold (renders at +1 pixel), styleItalic (via oblique param), styleUnderline; accumulates advance width

### sdl_font_info::_draw_text
- Signature: `int sdl_font_info::_draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool) const`
- Purpose: Public entry point to render bitmap font text to SDL surface
- Inputs: surface, text, length, position, color, style flags
- Outputs/Return: Rendered text width in pixels
- Side effects: Locks/unlocks surface; updates main screen dirty rect
- Calls: SDL_LockSurface(), ::draw_text<T>() (dispatched by BytesPerPixel), SDL_UnlockSurface(), MainScreenUpdateRect()
- Notes: Dispatches template draw_text based on surface color depth (1/2/4 bytes per pixel); respects draw_clip_rect_active; locks surface for pixel access

### ttf_font_info::_draw_text
- Signature: `int ttf_font_info::_draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool utf8) const`
- Purpose: Render TrueType font text to SDL surface
- Inputs: surface, text, length, position, color, style, utf8 flag
- Outputs/Return: Width of rendered text surface
- Side effects: Creates temporary SDL_Surface via SDL_ttf; blits to target; fills underline; updates main screen
- Calls: process_printable() or process_macroman(), TTF_RenderUTF8_Blended/Solid(), TTF_RenderUNICODE_Blended/Solid(), SDL_BlitSurface(), SDL_FillRect(), SDL_FreeSurface(), MainScreenUpdateRect()
- Notes: Supports UTF8 and MacRoman encodings; uses smooth (blended) or solid rendering per preferences; manually draws underline; respects clip_rect

### _draw_screen_text
- Signature: `void _draw_screen_text(const char *text, screen_rectangle *destination, short flags, short font_id, short text_color)`
- Purpose: High-level text drawing with alignment, clipping to destination, and automatic word wrapping
- Inputs: text, destination rect, alignment flags (_center_horizontal/_center_vertical/_right_justified/_top_justified/_wrap_text), font index, color index
- Outputs/Return: None
- Side effects: Recursive on wrap; calls draw_text which blits to draw_surface
- Calls: strncpy(), draw_text(), _draw_screen_text() (recursive for wrapped lines)
- Notes: Wraps on spaces; recursively draws overflow on next line; truncates if wider than destination; disables vertical centering on wrap; handles all four alignment modes

### set_drawing_clip_rectangle
- Signature: `void set_drawing_clip_rectangle(short top, short left, short bottom, short right)`
- Purpose: Set or clear the drawing clip rectangle
- Inputs: Clipping bounds (top < 0 clears clipping)
- Outputs/Return: None
- Side effects: Updates draw_clip_rect_active and draw_clip_rect globals
- Calls: None
- Notes: If top < 0, clipping is disabled

### _fill_rect / _frame_rect
- Signature: `void _fill_rect(screen_rectangle *rectangle, short color_index)` and `void _frame_rect(screen_rectangle *rectangle, short color_index)`
- Purpose: Fill or outline a rectangle with solid color
- Inputs: Rectangle (NULL fills screen), color index
- Outputs/Return: None
- Side effects: Fills/outlines in draw_surface; updates main screen
- Calls: _get_interface_color(), SDL_MapRGB(), SDL_FillRect() or draw_rectangle()
- Notes: _frame_rect draws 4 separate 1-pixel rectangles (top, bottom, left, right)

### draw_line
- Signature: `void draw_line(SDL_Surface *s, const world_point2d *v1, const world_point2d *v2, uint32 pixel, int pen_size)`
- Purpose: Draw line between two points with optional thickness and clipping
- Inputs: Surface, endpoints, color, pen size
- Outputs/Return: None
- Side effects: Locks/unlocks surface for thin lines; modifies pixel buffer directly or calls draw_polygon for thick
- Calls: draw_thin_line_noclip() or draw_polygon()
- Notes: Uses Cohen-Sutherland clipping; thin lines use DDA; thick lines rendered as 6-sided polygon

### draw_polygon
- Signature: `void draw_polygon(SDL_Surface *s, const world_point2d *vertex_array, int vertex_count, uint32 pixel)`
- Purpose: Render filled, clipped, convex polygon using scan-fill
- Inputs: Surface, vertex array, vertex count, color
- Outputs/Return: None
- Side effects: Allocates/reallocates static temporary arrays; fills polygon spans
- Calls: SDL_FillRect(), MainScreenUpdateRect()
- Notes: Uses Sutherland-Hodgman clipping; builds span list via edge tracing; handles screen clipping if draw_clip_rect_active

### Accessor functions (get_interface_rectangle, get_interface_color, get_interface_font, _get_interface_color, _get_player_color, _text_width, _get_font_line_height)
- Signature: Multiple
- Purpose: Retrieve interface resources by index with bounds checking; convert color formats
- Inputs: Index or pointer
- Outputs/Return: Pointer/reference to resource or converted color
- Side effects: None
- Calls: assert() for validation
- Notes: _get_interface_color/player_color map 16-bit colors to 8-bit via >>8 shift

## Control Flow Notes
Used in frame rendering for UI/HUD. Typical flow: initialize_screen_drawing() once at startup; _set_port_to_*() to choose target; then call _draw_screen_text/_draw_screen_shape/_fill_rect etc. multiple times per frame; _restore_port() before drawing to different surface.

## External Dependencies
- **SDL2**: SDL_Surface, SDL_Rect, SDL_LockSurface, SDL_BlitSurface, SDL_FillRect, SDL_FreeSurface, SDL_MapRGB, SDL_GetRGB
- **SDL2_ttf**: TTF_RenderUTF8_Blended/Solid, TTF_RenderUNICODE_Blended/Solid, TTF_FontAscent, TTF_FontHeight
- **Game headers**: shape_descriptors.h (get_shape_surface), screen.h (MainScreenSurface, MainScreenUpdateRects), sdl_fonts.h (font_info classes), FontHandler.h (FontSpecifier)
- **Utility**: process_printable(), process_macroman() (text encoding), get_ttf() (font lookup), environment_preferences (smooth_text flag)
