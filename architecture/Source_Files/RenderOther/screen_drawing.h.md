# Source_Files/RenderOther/screen_drawing.h

## File Purpose

Header file defining the screen drawing API for the Aleph One game engine. Declares functions and constants for rendering shapes, text, UI elements, and primitives to various drawing contexts. Manages a port-based drawing system for flexible rendering to screen, buffers, and specialized surfaces (HUD, terminal, map, intro).

## Core Responsibilities

- Define enumerated constants for interface rectangles (HUD elements, menu buttons, terminal screens), colors, fonts, and text justification flags
- Declare port/context management functions to set drawing destination (screen, gworld buffer, terminal, HUD, etc.)
- Declare shape and sprite rendering functions with support for clipping and centering
- Declare text rendering functions with measurement, wrapping, and justification options
- Declare screen manipulation functions (clear, scroll, fill, frame)
- Provide interface configuration queries (get rectangle bounds, color values, font objects)
- Declare low-level drawing primitives (polygon, line, rectangle rasterization)
- Provide inline wrapper helpers around `font_info` for text measurement and drawing

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `screen_rectangle` | struct | Portable rectangle (top, left, bottom, right); matches Mac `Rect` layout |
| `shape_descriptor` | typedef (uint16) | 16-bit handle: clut (3 bits) \| collection (5 bits) \| shape (8 bits); defined in shape_descriptors.h |
| `font_info` | class (abstract) | Base class for font handling; subclassed by `sdl_font_info` and `ttf_font_info`; defined in sdl_fonts.h |
| Rectangle ID enum | enum | 31 named constants for HUD rectangles, menu buttons, terminal screens |
| Color enum | enum | 23 named constants for weapon display, inventory, text, and computer interface colors |
| Font enum | enum | 7 named font types (interface, weapon name, player name, etc.) |
| Text justification enum | enum | 6 flags: center horizontal/vertical, right/top/bottom justified, wrap text |

## Global / File-Static State

None. Drawing state (current port, buffer pointers) is managed externally; this file is purely declarative.

## Key Functions / Methods

### initialize_screen_drawing
- **Signature:** `void initialize_screen_drawing(void);`
- **Purpose:** Initialization hook; sets up screen drawing subsystem
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes internal drawing state
- **Calls:** Not inferable
- **Notes:** Called during engine startup

### _set_port_to_* (port management)
- **Signature:** `void _set_port_to_screen_window(void);` and 5 variants (`_gworld`, `_term`, `_intro`, `_map`, `_custom(SDL_Surface *)`)
- **Purpose:** Select drawing destination (context)
- **Inputs:** `_set_port_to_custom` takes target SDL_Surface pointer
- **Outputs/Return:** None
- **Side effects:** Changes active drawing context; saves previous state for restoration
- **Calls:** Not inferable
- **Notes:** Must pair with `_restore_port()` to return to previous context

### _draw_screen_shape
- **Signature:** `void _draw_screen_shape(shape_descriptor shape_id, screen_rectangle *destination, screen_rectangle *source);`
- **Purpose:** Render sprite/shape to current port with optional source clipping
- **Inputs:** `shape_id` (collection+shape), `destination` (screen location), `source` (source clipping region; NULL = full shape)
- **Outputs/Return:** None
- **Side effects:** Modifies current drawing surface
- **Calls:** Sprite lookup/rendering (not visible)
- **Notes:** Source region permits partial sprite rendering

### _draw_screen_shape_at_x_y
- **Signature:** `void _draw_screen_shape_at_x_y(shape_descriptor shape, short x, short y);`
- **Purpose:** Convenience function; render sprite at pixel coordinates
- **Inputs:** `shape` descriptor, `x`, `y` screen position
- **Outputs/Return:** None
- **Side effects:** Modifies current drawing surface
- **Calls:** Likely delegates to `_draw_screen_shape` (not visible)

### _draw_screen_shape_centered
- **Signature:** `void _draw_screen_shape_centered(shape_descriptor shape, screen_rectangle *rectangle, short flags);`
- **Purpose:** Render sprite centered within a rectangle with layout flags
- **Inputs:** `shape`, `rectangle` (bounding box), `flags` (justification)
- **Outputs/Return:** None
- **Side effects:** Modifies current drawing surface
- **Calls:** Not inferable
- **Notes:** Supports horizontal/vertical centering via flags

### _draw_screen_text
- **Signature:** `void _draw_screen_text(const char *text, screen_rectangle *destination, short flags, short font_id, short text_color);`
- **Purpose:** Render text to current port with justification and styling
- **Inputs:** Text string, destination rectangle, justification flags, font ID, color index
- **Outputs/Return:** None
- **Side effects:** Modifies current drawing surface
- **Calls:** Font rendering functions (via sdl_fonts.h)
- **Notes:** Supports wrapping and multi-axis justification via flags

### _text_width (overloaded)
- **Signature:** `short _text_width(const char *buffer, short font_id);` and `short _text_width(const char *buffer, int start, int length, short font_id);`
- **Purpose:** Measure rendered text width in pixels
- **Inputs:** Text string, optional substring range, font ID
- **Outputs/Return:** Width in pixels
- **Side effects:** None
- **Calls:** Not inferable
- **Notes:** Variant supports substring measurement (start offset + length)

### _erase_screen
- **Signature:** `void _erase_screen(short color_index);`
- **Purpose:** Clear current drawing surface to solid color
- **Inputs:** Color index from color enum
- **Outputs/Return:** None
- **Side effects:** Fills entire current port with color
- **Calls:** Not inferable

### _scroll_window
- **Signature:** `void _scroll_window(short dy, short rectangle_id, short background_color_index);`
- **Purpose:** Scroll content within a named interface rectangle
- **Inputs:** Vertical delta pixels, rectangle ID, fill color for revealed area
- **Outputs/Return:** None
- **Side effects:** Scrolls pixels and fills background
- **Calls:** Not inferable
- **Notes:** Used for terminal/text window scrolling

### _fill_screen_rectangle / _fill_rect
- **Signature:** `void _fill_screen_rectangle(screen_rectangle *, short color_index);` and `void _fill_rect(screen_rectangle *, short color_index);`
- **Purpose:** Fill rectangle with solid color
- **Inputs:** Rectangle bounds, color index
- **Outputs/Return:** None
- **Side effects:** Modifies current drawing surface
- **Calls:** Not inferable
- **Notes:** Two function names suggest internal vs. interface variants

### _frame_rect
- **Signature:** `void _frame_rect(screen_rectangle *rectangle, short color_index);`
- **Purpose:** Draw rectangle outline (border only)
- **Inputs:** Rectangle bounds, color index
- **Outputs/Return:** None
- **Side effects:** Modifies current drawing surface
- **Calls:** Not inferable

### get_interface_rectangle / get_interface_color / get_interface_font / _get_font_line_height
- **Signature:** `screen_rectangle *get_interface_rectangle(short index);` / `rgb_color &get_interface_color(short index);` / `FontSpecifier &get_interface_font(short index);` / `short _get_font_line_height(short font_index);`
- **Purpose:** Query configuration data for interface elements
- **Inputs:** Index into rectangle/color/font arrays
- **Outputs/Return:** Pointer/reference to configuration struct; font line height
- **Side effects:** None
- **Calls:** Not inferable
- **Notes:** Support polygon/drawable queries via indexed enums

### draw_text / text_width / char_width / trunc_text (inline helpers)
- **Purpose:** Convenience wrappers around `font_info` virtual methods
- **Inputs:** SDL_Surface, text string, pixel color, font_info*, style flags, UTF-8 flag
- **Outputs/Return:** Pixel width or character width; truncation length
- **Side effects:** `draw_text` modifies SDL_Surface
- **Calls:** Delegate to `font_info::draw_text()`, `font_info::text_width()`, `font_info::char_width()`, `font_info::trunc_text()`
- **Notes:** All null-check font pointer and return 0 if null; reduce boilerplate throughout codebase

### draw_polygon / draw_line / draw_rectangle (extern primitives)
- **Signature:** `extern void draw_polygon(SDL_Surface *, const world_point2d *vertex_array, int vertex_count, uint32 pixel);` and similar for line/rectangle
- **Purpose:** Low-level rasterization of polygons, line segments, rectangles
- **Inputs:** Target surface, geometry, pixel color; line takes pen_size
- **Outputs/Return:** None
- **Side effects:** Modifies SDL_Surface
- **Calls:** Not defined in this file
- **Notes:** Defined elsewhere; used by higher-level drawing functions

## Control Flow Notes

Functions fit into the **render phase** of the game loop. The port-based design permits multiple rendering targets:
- **Screen**: Main game window  
- **Gworld**: Off-screen buffer (likely double-buffer for flicker-free rendering)  
- **HUD**: Dedicated buffer (mentioned as "always buffered" in header comment)  
- **Terminal/Intro/Map**: Specialized UI surfaces  

Typical sequence: `_set_port_to_X()` ΓåÆ draw calls ΓåÆ `_restore_port()`. The HUD buffer decouples UI rendering from main frame, enabling efficient compositing.

## External Dependencies

- **shape_descriptors.h** ΓÇö `shape_descriptor` type and collection/shape enumeration
- **sdl_fonts.h** ΓÇö `font_info`, `FontSpecifier` abstract font classes; font loading/unload
- **SDL2** ΓÇö `SDL_Surface`, `SDL_Rect`, `SDL_ttf.h` (implied by sdl_fonts.h dependency)
- **Defined elsewhere:** `rgb_color` (likely in color header), `world_point2d` (geometry header), `TextSpec` (font spec; used by sdl_fonts.h)
