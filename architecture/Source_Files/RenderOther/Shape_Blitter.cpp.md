# Source_Files/RenderOther/Shape_Blitter.cpp

## File Purpose
Implements `Shape_Blitter`, a utility class for rendering shape/bitmap data from the engine's shapes collection to 2D UI elements. Supports both OpenGL and SDL software rendering with transformations like scaling, rotation, tinting, and cropping.

## Core Responsibilities
- Load and cache shape surfaces from the shapes collection on-demand
- Manage surface memory (allocation, conversion to BGRA8888, deallocation)
- Rescale surfaces with automatic crop-rectangle adjustment
- Render shapes via OpenGL (with texture manager integration) or SDL (software blitter)
- Apply transformations: tinting (RGBA), rotation about center, cropping, mirroring
- Handle shape-type-specific rendering: walls (rotated), landscapes (vertically flipped), sprites, interface elements
- Track both original and scaled source dimensions for coordinate mapping

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Shape_Blitter | class | Main rendering utility; wraps a shape with transformations and dual rendering paths |
| Image_Rect | struct | Source/destination rectangles (x, y, w, h); used for positioning and cropping |
| SDL_Surface | typedef | SDL software surface (pixel data, format, palette) |
| shape_information_data | struct | Shape metadata (flags: mirroring, keypoint obscured, light intensity) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| shape_is_motion_blip | static function | file | Identifies motion sensor blips by collection/frame range; special transparency handling |

## Key Functions / Methods

### Shape_Blitter (Constructor)
- **Signature:** `Shape_Blitter(short collection, short frame_index, short texture_type, short clut_index)`
- **Purpose:** Initialize a shape blitter for a specific frame and texture type; load initial dimensions
- **Inputs:** collection (with embedded CLUT), frame index, texture type enum, CLUT index
- **Outputs/Return:** Object initialized, member variables set
- **Side effects:** Calls `get_shape_surface()` to load shape metadata; allocates/frees temporary SDL_Surface
- **Calls:** `get_shape_surface()`, `BUILD_COLLECTION()`, `SDL_FreeSurface()`, `free()`
- **Notes:** Does not load the full surface into `m_surface` yet (lazy-loaded on first draw); only caches dimensions

### Rescale
- **Signature:** `void Rescale(float width, float height)`
- **Purpose:** Resize the shape to new dimensions and adjust crop rectangle proportionally
- **Inputs:** target width, height (floats)
- **Outputs/Return:** void; updates `m_scaled_src` and `crop_rect` members
- **Side effects:** Modifies crop coordinates and dimensions
- **Calls:** (inline math)
- **Notes:** Maintains aspect ratio of crop region; used to fit shapes to UI layouts

### Width / Height
- **Signature:** `float Width()`, `float Height()`, `int UnscaledWidth()`, `int UnscaledHeight()`
- **Purpose:** Query current and original shape dimensions
- **Inputs:** none
- **Outputs/Return:** float or int dimension
- **Calls:** none
- **Notes:** Trivial accessors; scaled dimensions used for layout, unscaled for source lookups

### OGL_Draw
- **Signature:** `void OGL_Draw(const Image_Rect& dst)` (compiled only if `HAVE_OPENGL`)
- **Purpose:** Render shape using OpenGL; set up textures, apply transformations, issue draw commands
- **Inputs:** destination rectangle (screen coordinates)
- **Outputs/Return:** void; renders to OpenGL back buffer
- **Side effects:** Binds textures, modifies OpenGL state (modelview matrix, blend, sRGB framebuffer), calls `glDrawArrays()`
- **Calls:** `TextureManager::Setup()`, `extended_get_shape_bitmap_and_shading_table()`, `View_GetLandscapeOptions()`, `extended_get_shape_information()`, `OGL_RenderTexturedRect()`, OpenGL state functions
- **Notes:** 
  - Different branches for Interface, Landscape, and Sprite/Wall texture types; each applies unique UV coordinate transformations
  - Landscape branch swaps U/V axes and negates U scale
  - Sprite/Wall branch handles mirroring flags (_X_MIRRORED_BIT, _Y_MIRRORED_BIT)
  - Rotation applied via modelview matrix translate-rotate-translate pattern
  - sRGB framebuffer only enabled for non-weapon/non-HUD textures
  - Crop rect applied by adjusting texture offsets and scales

### SDL_Draw
- **Signature:** `void SDL_Draw(SDL_Surface *dst_surface, const Image_Rect& dst)`
- **Purpose:** Render shape using SDL software blitter; lazy-load surface, rescale, apply type-specific transforms
- **Inputs:** destination SDL_Surface, destination rectangle
- **Outputs/Return:** void; blits to `dst_surface`
- **Side effects:** On first call, loads and converts shape surface; caches in `m_surface`; may allocate `m_scaled_surface`; calls `SDL_BlitSurface()`
- **Calls:** `get_shape_surface()`, `SDL_ConvertSurfaceFormat()`, `SDL_SetColorKey()`, `SDL_FreeSurface()`, `shape_is_motion_blip()`, `rescale_surface()`, `rotate_surface()`, `flip_surface()`, `SDL_BlitSurface()`
- **Notes:**
  - Surface caching: `m_surface` loaded once, `m_scaled_surface` regenerated only if dimensions change
  - Wall textures rotated post-scale; landscape textures flipped post-scale
  - Motion blips get special color-key transparency handling
  - Crop rect applied as source blit rectangle; destination scaled by `dst.w` / `dst.h`

### flip_surface (static helper)
- **Signature:** `SDL_Surface* flip_surface(SDL_Surface *s, int width, int height)`
- **Purpose:** Vertically flip a surface by reversing scanline order
- **Inputs:** source surface, output dimensions
- **Outputs/Return:** New SDL_Surface (flipped); caller must free
- **Side effects:** Allocates new surface; copies/reverses pixel data; copies palette if present
- **Calls:** `SDL_CreateRGBSurface()`, `SDL_SetPaletteColors()`, `SDL_FreeSurface()` (by caller)
- **Notes:** Preserves pixel format and palette; used for landscape rendering

### Destructor
- **Signature:** `~Shape_Blitter()`
- **Purpose:** Clean up cached surfaces
- **Outputs/Return:** void
- **Side effects:** Frees `m_surface` and `m_scaled_surface` (if distinct)
- **Calls:** `SDL_FreeSurface()`

## Control Flow Notes
Shape_Blitter is a transient rendering helper created during UI/HUD draw passes:
1. **Construction:** Load shape metadata; defer surface load
2. **Optional rescale:** Adjust dimensions if needed for layout
3. **Render:** Call `OGL_Draw()` or `SDL_Draw()` based on renderer (lazy-loads surface on first draw)
4. **Destruction:** Free cached surfaces

The class supports dual rendering (OpenGL and SDL) selected at runtime by the caller. OpenGL path uses the TextureManager for advanced features (glow maps, sRGB); SDL path is simpler software blitting with type-specific CPU transforms.

## External Dependencies
- **Shape system:** `get_shape_surface()`, `extended_get_shape_information()`, `extended_get_shape_bitmap_and_shading_table()`, `shapes_file_is_m1()` ΓÇö load shape data and metadata
- **OpenGL texture manager:** `TextureManager`, `View_GetLandscapeOptions()`, `OGL_RenderTexturedRect()`, OpenGL headers
- **SDL:** `SDL_Surface`, `SDL_ConvertSurfaceFormat()`, `SDL_BlitSurface()`, surface creation/manipulation
- **Rendering config:** `Wanting_sRGB`, `Using_sRGB`, OGL setup/teardown globals
- **Image utilities:** `rescale_surface()`, `rotate_surface()`, `tile_surface()` (from images.h)
- **Shape definitions:** `shape_descriptor`, `shape_information_data`, texture type enums from scotish_textures.h and interface.h
