# Source_Files/RenderOther/FontHandler.h

## File Purpose
Defines the `FontSpecifier` class for font specification and rendering in Aleph One (a Marathon-like game engine). Provides parameterized font configuration (size, style, file), metric computation, and unified text rendering for both SDL and OpenGL backends with OpenGL texture/display-list management.

## Core Responsibilities
- Store and manage font parameters (size, style, filename, name set)
- Compute and cache derived font metrics (height, ascent, descent, line spacing, character widths)
- Initialize and update fonts from parameters (called by XML parser)
- Render text via SDL surfaces or OpenGL with position/alignment control
- Manage OpenGL font textures, display lists, and filtering parameters
- Track active OpenGL fonts in a static registry for context-switch cleanup
- Provide character and text width queries for layout calculations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FontSpecifier` | class | Main font specification and rendering interface |
| `font_info` | class (abstract, from sdl_fonts.h) | Font backend abstraction for SDL/TTF rendering |
| `screen_rectangle` | struct (fwd-decl) | Rectangle for text bounds and alignment |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_font_registry` | `std::set<FontSpecifier*>*` | static (class member) | Tracks all active OpenGL font instances for context-switch resource cleanup |

## Key Functions / Methods

### Init()
- **Signature:** `void Init();`
- **Purpose:** Initialize font object before use (workaround for lack of proper constructor)
- **Inputs:** None (uses `this->` members)
- **Outputs/Return:** None
- **Side effects:** Initializes internal state; likely calls `Update()` internally
- **Calls:** Presumably `Update()`
- **Notes:** Must be called before any other method

### Update()
- **Signature:** `void Update();`
- **Purpose:** Recompute derived font metrics from parameter fields
- **Inputs:** `Size`, `Style`, `AdjustLineHeight`, `File` (member fields)
- **Outputs/Return:** Populates `Height`, `LineSpacing`, `Ascent`, `Descent`, `Leading`, `Widths[256]`, `Info`
- **Side effects:** Loads/reloads font backend; may allocate `Info`
- **Calls:** Font backend loading (via `load_font()` or similar)
- **Notes:** Called by `Init()` and XML parser when parameters change

### TextWidth(const char *Text)
- **Signature:** `int TextWidth(const char *Text);`
- **Purpose:** Compute total pixel width of a C-style string
- **Inputs:** Text (null-terminated C string)
- **Outputs/Return:** Width in pixels
- **Side effects:** None
- **Calls:** Delegates to `Info->text_width()` or similar
- **Notes:** Used for centering layout (map titles)

### CharWidth(char c)
- **Signature:** `int CharWidth(char c) const;`
- **Purpose:** Look up pixel width of a single character
- **Inputs:** Character `c`
- **Outputs/Return:** Width from `Widths[static_cast<int>(c)]`
- **Side effects:** None
- **Calls:** None (inline table lookup)
- **Notes:** Fast array lookup; assumes char is valid index [0..255]

### OGL_Reset(bool IsStarting)
- **Signature:** `void OGL_Reset(bool IsStarting);`
- **Purpose:** Reset OpenGL font texture and display list state (context change or initialization)
- **Inputs:** `IsStarting` (true for context creation, false for context recreation)
- **Outputs/Return:** None
- **Side effects:** Deallocates/reallocates `OGL_Texture`, `TxtrID`, `DispList`; avoids memory leaks
- **Calls:** OpenGL functions (glGenTextures, glDeleteLists, etc.)
- **Notes:** Guard with `#ifdef HAVE_OPENGL`; context switches require cleanup

### OGL_Render(const char *Text)
- **Signature:** `void OGL_Render(const char *Text);`
- **Purpose:** Render text in OpenGL at screen coordinates with modelview matrix updates
- **Inputs:** Text (null-terminated C string)
- **Outputs/Return:** None
- **Side effects:** Alters OpenGL modelview matrix; advances position for next characters
- **Calls:** OpenGL rendering commands (glTranslate, glCallList, etc.)
- **Notes:** Assumes left baseline at (0,0); users can wrap with glPushMatrix/glPopMatrix for restoration; guard with `#ifdef HAVE_OPENGL`

### OGL_DrawText(const char *Text, const screen_rectangle &r, short flags)
- **Signature:** `void OGL_DrawText(const char *Text, const screen_rectangle &r, short flags);`
- **Purpose:** Render text in OpenGL with alignment and wrapping (high-level API)
- **Inputs:** Text, bounding rectangle, alignment/wrap flags
- **Outputs/Return:** None
- **Side effects:** Renders to framebuffer; modelview matrix preserved
- **Calls:** `OGL_Render()`, text layout logic
- **Notes:** Guard with `#ifdef HAVE_OPENGL`; similar to `_draw_screen_text()` from screen_drawing.h

### OGL_ResetFonts(bool IsStarting) [static]
- **Signature:** `static void OGL_ResetFonts(bool IsStarting);`
- **Purpose:** Reset all active fonts in the registry (bulk context-switch cleanup)
- **Inputs:** `IsStarting`
- **Outputs/Return:** None
- **Side effects:** Iterates `m_font_registry`, calls `OGL_Reset()` on each
- **Calls:** `OGL_Reset()` on registry members
- **Notes:** Guard with `#ifdef HAVE_OPENGL`; called on context creation/recreation

### OGL_Register/Deregister(FontSpecifier *F) [static]
- **Signature:** `static void OGL_Register(FontSpecifier *F);` and `static void OGL_Deregister(FontSpecifier *F);`
- **Purpose:** Add/remove font instance from the global registry for resource lifecycle tracking
- **Inputs:** Pointer to this font instance
- **Outputs/Return:** None
- **Side effects:** Insert/erase from `m_font_registry`; likely called in constructor/destructor
- **Calls:** `std::set::insert()`, `std::set::erase()`
- **Notes:** Guard with `#ifdef HAVE_OPENGL`; essential for clean context switching

### DrawText(SDL_Surface *s, const char *text, int x, int y, uint32 pixel, bool utf8)
- **Signature:** `int DrawText(SDL_Surface *s, const char *text, int x, int y, uint32 pixel, bool utf8 = false);`
- **Purpose:** Unified API for rendering text on SDL surface (backend-agnostic)
- **Inputs:** Surface, text, position (x, y), pixel color, UTF-8 flag
- **Outputs/Return:** Likely bytes drawn or final x position
- **Side effects:** Modifies SDL surface pixels
- **Calls:** `Info->draw_text()`
- **Notes:** Works regardless of OpenGL availability; UTF-8 support optional

### operator==, operator!=, operator=
- **Signature:** `bool operator==(FontSpecifier& F);`, `bool operator!=(FontSpecifier& F);`, `FontSpecifier& operator=(FontSpecifier& F);`
- **Purpose:** Equality comparison and assignment
- **Inputs:** Another `FontSpecifier&`
- **Outputs/Return:** bool (comparison) or `FontSpecifier&` (assignment)
- **Side effects:** Assignment may copy/reallocate resources
- **Calls:** None visible
- **Notes:** Comparison likely checks parameters; assignment likely deep-copies `Info`

### ~FontSpecifier() [destructor]
- **Signature:** `~FontSpecifier();`
- **Purpose:** Clean up font resources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deallocates `Info`, `OGL_Texture`, display list; deregisters from `m_font_registry`
- **Calls:** `unload_font()`, `OGL_Deregister()`, `glDelete*`

## Control Flow Notes
- **Initialization phase:** `Init()` called once before use; sets up font metrics and backend (SDL/TTF).
- **Configuration phase:** `Update()` called when XML parser changes `Size`, `Style`, `File`, or `AdjustLineHeight`.
- **OpenGL context lifecycle:** `OGL_Reset(true)` on context creation, `OGL_Reset(false)` on context recreation; `OGL_ResetFonts()` called globally on context switches.
- **Render frame:** `OGL_Render()` or `OGL_DrawText()` called per frame for text output; `DrawText()` for SDL fallback.
- **Resource cleanup:** Destructor or explicit deregistration removes font from `m_font_registry` on shutdown.

## External Dependencies
- **cseries.h:** Core types (`uint8`, `uint16`, `uint32`, `int16`, `short`), macros, basic utilities
- **sdl_fonts.h:** Abstract `font_info` class, SDL/TTF font loading (`load_font()`, `unload_font()`), `TextSpec` struct, font styles (`styleNormal`, `styleBold`, `styleItalic`, `styleUnderline`)
- **OGL_Headers.h:** OpenGL headers (conditional on `HAVE_OPENGL`); provides `GLuint`, `GL_LINEAR`, etc.
- **SDL2/SDL.h:** Via cseries.h; `SDL_Surface` type
- **Standard library:** `std::string`, `std::set`
- **screen_drawing.h:** Referenced for `_draw_screen_text()` behavior model (not included directly)
