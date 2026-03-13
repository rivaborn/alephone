# Source_Files/RenderOther/OGL_Blitter.h

## File Purpose
Declares `OGL_Blitter`, an OpenGL-based image blitter that inherits from `Image_Blitter` to render 2D images to screen. Manages OpenGL texture creation, tiling across multiple 256├ù256 tiles, and texture lifecycle on context switches via a static registry of active blitters.

## Core Responsibilities
- Create, load, and unload OpenGL textures for image blitting
- Provide `Draw()` method overloads to render images with source/destination rectangles
- Tile large images across multiple 256├ù256 textures and track them in `m_refs`
- Maintain a static registry (`m_blitter_registry`) to track all active blitters for safe context switching
- Expose static screen/viewport management: binding, coordinate conversion, dimension queries
- Support configurable texture filtering via `nearFilter` parameter

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Blitter` | class | Inherits `Image_Blitter`; OpenGL-backed image blitter |
| `m_rects` | `vector<SDL_Rect>` | Rectangle regions for each tile (tiling layout) |
| `m_refs` | `vector<GLuint>` | OpenGL texture handles for each tile |
| `m_blitter_registry` | `static std::set<OGL_Blitter*>` | Registry of all active blitters for context-switch recovery |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_blitter_registry` | `std::set<OGL_Blitter*>*` | static | Tracks all in-use OGL_Blitter instances to reload textures on context loss |
| `tile_size` | `static const int` | static | Tile dimension (256 pixels); divides images into square tiles for rendering |

## Key Functions / Methods

### OGL_Blitter (constructor)
- Signature: `OGL_Blitter(GLuint nearFilter = GL_LINEAR)`
- Purpose: Construct an OpenGL blitter with optional texture filtering mode
- Inputs: `nearFilter` ΓÇô OpenGL minification/magnification filter enum (default `GL_LINEAR`)
- Outputs/Return: ΓÇö
- Side effects: Registers this instance in `m_blitter_registry`
- Calls: `Register(this)`

### Unload
- Signature: `void Unload()`
- Purpose: Release GPU resources (textures, etc.)
- Inputs: ΓÇö
- Outputs/Return: ΓÇö
- Side effects: Calls `_UnloadTextures()`; may deallocate GLuint handles
- Calls: `_UnloadTextures()`

### Draw (three overloads)
- Signature:
  - `void Draw(SDL_Surface *dst_surface, const Image_Rect& dst, const Image_Rect& src)`
  - `void Draw(SDL_Surface *dst_surface, const Image_Rect& dst)` ΓåÆ calls above with `crop_rect`
  - `void Draw(const Image_Rect& dst, const Image_Rect& src)` ΓåÆ primary implementation
- Purpose: Render the image to a destination rectangle with optional source clipping
- Inputs: destination rect (`dst`), source rect (`src`), optional SDL surface target
- Outputs/Return: ΓÇö
- Side effects: OpenGL rendering calls; may load textures on first use
- Calls: Other `Draw()` overloads; `_LoadTextures()`

### Static Screen / Viewport Methods
- `StopTextures()` ΓÇô disables all textures globally (context switch cleanup)
- `BoundScreen(bool in_game)` ΓÇô binds OpenGL rendering to screen; `in_game` flag controls viewport
- `WindowToScreen(int& x, int& y)` ΓÇô converts window coordinates to screen space (in-place)
- `ScreenWidth()`, `ScreenHeight()` ΓÇô query current rendering surface dimensions
- Notes: These are static utility functions for global GL state and coordinate mapping

### Private Texture Management
- `_LoadTextures()` ΓÇô creates OpenGL texture objects and populates `m_refs`, `m_rects` based on image dimensions
- `_UnloadTextures()` ΓÇô frees GLuint handles
- `Register()`, `Deregister()` ΓÇô add/remove this instance from static `m_blitter_registry` for context recovery

## Control Flow Notes
- **Initialization**: Constructor registers instance; first `Draw()` call triggers `_LoadTextures()` on demand
- **Rendering**: `Draw()` with source/dest rects issues OpenGL commands; may span multiple texture tiles
- **Context Loss**: `StopTextures()` called globally; static registry allows each blitter to reload on `_LoadTextures()` next frame
- **Shutdown**: Destructor calls `Unload()` and deregisters from `m_blitter_registry`

## External Dependencies
- `cseries.h` ΓÇô platform/utility macros
- `Image_Blitter.h` ΓÇô base class with `crop_rect`, tint, rotation, surface management
- `ImageLoader.h` ΓÇô `ImageDescriptor` type (not directly used in header, but implied for image loading)
- `OGL_Headers.h` ΓÇô OpenGL typedefs (`GLuint`), platform-specific GL setup
- `<vector>`, `<set>` ΓÇô STL containers
- `SDL_Surface`, `SDL_Rect` ΓÇô SDL2 types
