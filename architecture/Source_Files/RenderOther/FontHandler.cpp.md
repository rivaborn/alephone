# Source_Files/RenderOther/FontHandler.cpp

## File Purpose
Implements font specification and rendering for both SDL and OpenGL backends. Manages font metrics, precalculates glyph widths, generates OpenGL font textures with display lists, and provides text rendering with alignment and wrapping support.

## Core Responsibilities
- Font specification management (size, style, file parsing with special ID handling)
- Font metric extraction (ascent, descent, leading, line spacing)
- Per-character width precalculation for all 256 character codes
- OpenGL texture atlas generation from SDL surface rendering
- OpenGL display list creation for efficient glyph rendering
- Text rendering with horizontal/vertical alignment and word wrapping
- Registry management for font resources across OpenGL context switches
- Dual-mode rendering (SDL vs. OpenGL) abstraction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FontSpecifier | class | Main font handler; manages all font state and rendering |
| TextSpec | struct | Intermediate font specification passed to `load_font()` |
| font_info | struct | SDL font info (defined elsewhere); holds glyph metrics |
| screen_rectangle | struct | Portable rectangle for text layout bounds |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FontSpecifier::m_font_registry` | `std::set<FontSpecifier*>*` | static | Registry of all active FontSpecifier instances for coordinated OpenGL cleanup on context switch |

## Key Functions / Methods

### FontSpecifier::Init()
- **Signature:** `void Init()`
- **Purpose:** Initialize a font specifier before use; required due to lack of proper constructor
- **Inputs:** None (uses member variables)
- **Outputs/Return:** None
- **Side effects:** Sets `Info = NULL`, calls `Update()`, initializes `OGL_Texture = NULL` (if OpenGL)
- **Calls:** `Update()`, `OGL_Reset(false)` (destructor cleanup)
- **Notes:** Must be called before any font rendering

### FontSpecifier::Update()
- **Signature:** `void Update()`
- **Purpose:** Load font resource and precalculate all metrics and glyph widths
- **Inputs:** `Size`, `Style`, `File` (member variables); `AdjustLineHeight`
- **Outputs/Return:** None
- **Side effects:** Unloads old font, loads new one via `load_font()`, fills `Widths[256]`, sets `Ascent`, `Descent`, `Leading`, `Height`, `LineSpacing`
- **Calls:** `unload_font()`, `load_font()`, `char_width()` (256 times)
- **Notes:** Special handling for font IDs: `#4` ΓåÆ Monaco (1.34├ù scale), `#22` ΓåÆ Courier Prime variants with height adjustment. Falls back to direct filename if not a `#ID` format.

### FontSpecifier::TextWidth()
- **Signature:** `int TextWidth(const char *text)`
- **Purpose:** Calculate total pixel width of a text string
- **Inputs:** C-string (null-terminated); safe on NULL input
- **Outputs/Return:** Summed width in pixels
- **Side effects:** None
- **Calls:** (inline access to `Widths[256]`)
- **Notes:** Uses precalculated `Widths` array; O(n) in string length

### FontSpecifier::OGL_Reset(bool IsStarting)
- **Signature:** `void OGL_Reset(bool IsStarting)`
- **Purpose:** Create/destroy OpenGL font texture and display lists; called on context switch or initialization
- **Inputs:** `IsStarting = true` for init, `false` for cleanup
- **Outputs/Return:** None
- **Side effects:** If `IsStarting`: allocates `OGL_Texture`, `TxtrID`, `DispList`, sets `TxtrWidth`/`TxtrHeight`. If not: deletes GL resources and deregisters from global registry
- **Calls:** `SDL_CreateRGBSurface()`, `draw_text()` (SDL), `glGenTextures()`, `glGenLists()`, `glNewList()`, `glEndList()`, `OGL_RenderTexturedRect()`, `OGL_Register()`, `OGL_Deregister()`, `NextPowerOfTwo()`, `MAX()`
- **Notes:** Pads glyphs by 1 pixel to avoid clipping. Arranges all 256 glyphs in a single power-of-two texture. Early return if font is empty. Display lists record matrix translations and `OGL_RenderTexturedRect()` calls for each glyph.

### FontSpecifier::OGL_Render(const char *Text)
- **Signature:** `void OGL_Render(const char *Text)`
- **Purpose:** Render text string using OpenGL display lists; assumes screen coordinates with baseline at (0,0)
- **Inputs:** C-string; up to 255 chars (truncated)
- **Outputs/Return:** None
- **Side effects:** Alters modelview matrix; sets texture binding, blend mode, attributes
- **Calls:** `OGL_Reset(true)`, `glPushAttrib()`, `glEnable()`, `glDisable()`, `glBlendFunc()`, `glBindTexture()`, `glCallList()`, `glPopAttrib()`
- **Notes:** Lazy init: if no texture, calls `OGL_Reset(true)`. Each `glCallList()` advances modelview matrix for next glyph.

### FontSpecifier::OGL_DrawText(const char *text, const screen_rectangle &r, short flags)
- **Signature:** `void OGL_DrawText(const char *text, const screen_rectangle &r, short flags)`
- **Purpose:** Render text with alignment and word wrapping; modelview matrix preserved
- **Inputs:** Text, rectangle bounds, flags (`_center_horizontal`, `_center_vertical`, `_right_justified`, `_top_justified`, `_wrap_text`)
- **Outputs/Return:** None
- **Side effects:** Calls `OGL_Render()` within matrix push/pop
- **Calls:** `strlen()`, `TextWidth()`, `CharWidth()`, `OGL_DrawText()` (recursive on wrap), `glMatrixMode()`, `glPushMatrix()`, `glTranslated()`, `OGL_Render()`, `glPopMatrix()`
- **Notes:** Recursive word wrapping: if text exceeds width, wraps at last space and recursively calls itself on next line with `_top_justified` set. Truncates mid-word if no space found.

### FontSpecifier::OGL_ResetFonts(bool IsStarting) [static]
- **Signature:** `static void OGL_ResetFonts(bool IsStarting)`
- **Purpose:** Reset all registered FontSpecifier instances (engine-level context switch handler)
- **Inputs:** `IsStarting` flag
- **Outputs/Return:** None
- **Side effects:** Iterates `m_font_registry`, calls `OGL_Reset()` on each
- **Calls:** `OGL_Reset()` on each registered font
- **Notes:** When starting, iterates forward. When stopping, iterates from begin each time (iterator-safe deletion).

### FontSpecifier::OGL_Register / OGL_Deregister [static]
- **Signature:** `static void OGL_Register(FontSpecifier *F)` / `static void OGL_Deregister(FontSpecifier *F)`
- **Purpose:** Add/remove font from global registry for coordinated cleanup
- **Inputs:** Pointer to FontSpecifier
- **Outputs/Return:** None
- **Side effects:** Lazily allocates `m_font_registry` if needed; inserts/erases pointer
- **Calls:** `new std::set<>`, `insert()`, `erase()`

### FontSpecifier::DrawText(SDL_Surface *s, const char *text, int x, int y, uint32 pixel, bool utf8)
- **Signature:** `int DrawText(SDL_Surface *s, const char *text, int x, int y, uint32 pixel, bool utf8 = false)`
- **Purpose:** Render text to SDL surface or OpenGL screen, abstracted from backend
- **Inputs:** Destination surface (NULL safe), text, position, pixel color, UTF-8 flag
- **Outputs/Return:** 1 if rendered (OpenGL), 0 if SDL or error
- **Side effects:** If OpenGL: sets color, renders to both buffers, swaps buffers
- **Calls:** `MainScreenSurface()`, `MainScreenIsOpenGL()`, `draw_text()` (SDL), `SDL_GetRGB()`, `glColor4ub()`, `glMatrixMode()`, `glPushMatrix()`, `glTranslatef()`, `OGL_Render()`, `glPopMatrix()`, `MainScreenSwap()`
- **Notes:** Returns to caller after first buffer in OpenGL mode (loops twice internally). If main screen is OpenGL, routes to `OGL_Render()`; otherwise calls SDL `draw_text()`.

### FontSpecifier::operator==, operator=, ~FontSpecifier
- **Equality**: Compares `Size`, `Style`, `File` (not derived metrics)
- **Assignment**: Copies all three parameters
- **Destructor**: Calls `OGL_Reset(false)` if OpenGL enabled

## Control Flow Notes
- **Initialization**: `Init()` ΓåÆ `Update()` ΓåÆ `OGL_Reset(true)` (on first draw if needed)
- **Rendering loop**: `OGL_Render()` or `DrawText()` called per frame; display lists reused
- **Context switch**: Engine calls `OGL_ResetFonts(false)` ΓåÆ cleans up all fonts ΓåÆ `OGL_ResetFonts(true)` ΓåÆ regenerates all
- **Shutdown**: Destructor ΓåÆ `OGL_Reset(false)` per instance

## External Dependencies
- **SDL**: `SDL_Surface`, `SDL_CreateRGBSurface()`, `SDL_FillRect()`, `SDL_MapRGB()`, `SDL_FreeSurface()`, `SDL_GetRGB()`
- **OpenGL**: `glGenTextures()`, `glBindTexture()`, `glTexEnvi()`, `glTexParameteri()`, `glTexImage2D()`, `glGenLists()`, `glNewList()`, `glTranslatef()`, `glCallList()`, `glColor4ub()`, `glMatrixMode()`, `glPushMatrix()`, `glPopMatrix()`, `glEnable()`, `glDisable()`, etc.
- **Engine**: `load_font()`, `unload_font()`, `draw_text()` (SDL), `char_width()` (from sdl_fonts / screen_drawing.cpp)
- **Engine**: `OGL_RenderTexturedRect()`, `OGL_Register()`, `OGL_Deregister()`, `MainScreenSurface()`, `MainScreenIsOpenGL()`, `MainScreenSwap()`
- **Engine**: `NextPowerOfTwo()`, `MAX()`, `MIN()` (macros/utilities)
- **Standard**: `<math.h>`, `<string.h>`, `std::set`, `std::string`
