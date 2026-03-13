# Source_Files/RenderOther/OGL_Blitter.cpp

## File Purpose
Implements efficient 2D image rendering via OpenGL by tiling SDL surfaces into power-of-two GPU textures. Manages texture lifecycle and provides lazy-loaded rendering with support for scaling, rotation, and per-instance tinting.

## Core Responsibilities
- Decompose source images into tile-aligned chunks matching GPU constraints
- Load SDL surface pixels into OpenGL 2D textures with configurable filtering
- Render tiled texture rectangles with coordinate transformation and clipping
- Maintain a global registry of active blitters for coordinated cleanup on GL context loss
- Handle edge-pixel smearing to eliminate texture sampling artifacts at tile boundaries
- Support view transformations (scaling, rotation) in the render pipeline

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OGL_Blitter | class | Inherits Image_Blitter; manages tiled GL texture rendering |
| Image_Rect | struct | Rectangle {x, y, w, h} for source/destination regions |
| SDL_Rect | struct | SDL rectangle for pixel blitting and tile bounds |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| OGL_Blitter::tile_size | static const int | class static | Maximum tile dimension; set to 256 |
| OGL_Blitter::m_blitter_registry | static std::set<OGL_Blitter*>* | class static | Registry of active instances for bulk cleanup on GL context switch |

## Key Functions / Methods

### _LoadTextures
- Signature: `void _LoadTextures()`
- Purpose: Convert source SDL surface into tiled OpenGL 2D textures
- Inputs: Uses inherited m_surface, m_src, m_scaled_src; reads OGL_Flag_TextureFix from config
- Outputs/Return: Populates m_rects (tile rectangles), m_refs (GL texture IDs); sets m_textures_loaded=true
- Side effects: Allocates GPU memory via glTexImage2D; enables GL_TEXTURE_2D; calls Register(); modifies pixel data (edge smearing)
- Calls: NextPowerOfTwo(), Get_OGL_ConfigureData(), SDL_CreateRGBSurface(), SDL_BlitSurface(), PlatformIsLittleEndian(), glGenTextures(), glTexParameteri(), glTexImage2D(), glBindTexture()
- Notes: Early exit if already loaded or m_surface absent. Tile width/height are min(NextPowerOfTwo(dim), 256), with optional floor of 128 if TextureFix enabled. Edge pixels are replicated horizontally and vertically to prevent seam artifacts.

### Draw
- Signature: `void Draw(const Image_Rect& dst, const Image_Rect& raw_src)`
- Purpose: Render a source region to a destination screen rectangle with optional rotation and tinting
- Inputs: dst (screen destination), raw_src (logical source in scaled-space coordinates)
- Outputs/Return: None
- Side effects: Lazy-triggers _LoadTextures(); modifies GL state (attributes, blend, matrix, texture binding, color); renders quads to framebuffer
- Calls: _LoadTextures(), glPushAttrib(), glDisable(), glEnable(), glBlendFunc(), glColor4f(), glMatrixMode(), glPushMatrix(), glTranslatef(), glRotatef(), glBindTexture(), OGL_RenderTexturedRect(), glPopMatrix(), glPopAttrib()
- Notes: Transforms raw_src coordinates from scaled-space to source-space if m_src Γëá m_scaled_src. Sets up orthogonal projection with rotation pivot at dst center. Frustum-culls tiles outside raw_src bounds. Computes normalized texture coords (V/U min/max) per tile and blends with GL_SRC_ALPHA / GL_ONE_MINUS_SRC_ALPHA. Uses inherited rotation and tint_color_{r,g,b,a} members.

### _UnloadTextures
- Signature: `void _UnloadTextures()`
- Purpose: Release GPU texture resources and deregister instance
- Inputs: None
- Outputs/Return: None
- Side effects: Calls glDeleteTextures(); clears m_rects and m_refs; calls Deregister(); sets m_textures_loaded=false
- Calls: glDeleteTextures(), Deregister()
- Notes: Safe to call multiple times (guarded by m_textures_loaded check)

### StopTextures (static)
- Signature: `static void StopTextures()`
- Purpose: Force-unload textures for all registered blitters (used on GL context loss/recreation)
- Inputs: None
- Outputs/Return: None
- Side effects: Iterates registry, calling _UnloadTextures() on each blitter
- Calls: _UnloadTextures() (via pointer dereference)
- Notes: Loop re-fetches registry.begin() after each removal to handle concurrent modification

### Register / Deregister (static)
- Signature: `static void Register(OGL_Blitter *B)`, `static void Deregister(OGL_Blitter *B)`
- Purpose: Track active blitter instances in global registry
- Inputs: Pointer to OGL_Blitter
- Outputs/Return: None
- Side effects: Lazily allocates m_blitter_registry if null; inserts/erases from set
- Calls: None
- Notes: Used to ensure StopTextures() can find all instances without manual tracking

### Accessor / Delegator Methods
- `ScreenWidth()`, `ScreenHeight()` ΓÇô delegate to MainScreenLogicalWidth()/Height()
- `BoundScreen()`, `WindowToScreen()` ΓÇô delegate to alephone::Screen::instance() methods
- `Unload()` ΓÇô calls parent Image_Blitter::Unload() then _UnloadTextures()
- `~OGL_Blitter()` ΓÇô destructor; calls Unload()

## Control Flow Notes
**Lazy Loading**: Textures created on first Draw() call, not in constructor, allowing deferred resource allocation.

**Render Flow**: Each Draw() checks Loaded() (parent class), then _LoadTextures() if needed, then iterates m_rects to render visible tiles. Rotation and scaling are applied per-invocation via GL matrix stack.

**Lifecycle**: Register() called during _LoadTextures(); StopTextures() called (externally) on GL context loss to prevent resource leaks. Deregister() called during _UnloadTextures().

## External Dependencies
- **OGL_Setup.h** ΓÇô Get_OGL_ConfigureData(), PlatformIsLittleEndian(), OpenGL constants
- **OGL_Render.h** ΓÇô OGL_RenderTexturedRect() helper
- **screen.h** ΓÇô alephone::Screen singleton, MainScreenLogicalWidth/Height()
- **shell.h** ΓÇô shell utilities
- **&lt;memory&gt;** ΓÇô std::unique_ptr with SDL_FreeSurface deleter
- **SDL2** ΓÇô SDL_Surface, SDL_Rect, SDL_CreateRGBSurface(), SDL_BlitSurface(), SDL_SetSurfaceBlendMode()
- **OpenGL (global)** ΓÇô glGenTextures(), glBindTexture(), glTexParameteri(), glTexImage2D(), glDeleteTextures(), GL_TEXTURE_2D, blend/matrix ops
- **Parent class Image_Blitter** ΓÇô provides Loaded(), m_surface, m_src, m_scaled_src, crop_rect, rotation, tint_color_{r,g,b,a}
