# Source_Files/RenderOther/Image_Blitter.cpp

## File Purpose
Implements the `Image_Blitter` class for rendering 2D images in the game UI. Manages SDL surface lifecycle, image loading from multiple sources, rescaling, and drawing with support for tinting and rotation.

## Core Responsibilities
- Load images from `ImageDescriptor` objects, resource IDs, or SDL_Surface objects
- Manage SDL surface memory (creation, conversion, rescaling, cleanup)
- Maintain image state: source dimensions, scaled dimensions, and crop rectangle
- Handle on-demand surface conversion and rescaling during draw operations
- Provide dimension queries (scaled and unscaled)
- Draw images to destination surfaces with optional source cropping

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Image_Rect` | struct | Float-based rectangle for destination/source regions |
| `Image_Blitter` | class | Main image rendering wrapper managing SDL surfaces |
| `SDL_Surface` | external | SDL2 pixel data container |
| `SDL_Rect` | external | SDL2 integer rectangle |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor
- Signature: `Image_Blitter()`
- Purpose: Initialize all member variables to default/null state
- Inputs: None
- Outputs: Object instance
- Side effects: Allocates Image_Blitter instance with all surfaces set to NULL
- Calls: None (pure initialization)
- Notes: Tint color defaults to (1.0, 1.0, 1.0, 1.0); rotation defaults to 0.0

### Load (ImageDescriptor variant)
- Signature: `bool Load(const ImageDescriptor& image)`
- Purpose: Load image data from an ImageDescriptor, handling endianness conversion
- Inputs: `image` ΓÇô ImageDescriptor with pixel buffer and dimensions
- Outputs: `true` if successful, `false` otherwise
- Side effects: Creates temporary SDL_Surface, calls `Load(SDL_Surface&)`
- Calls: `PlatformIsLittleEndian()`, `SDL_CreateRGBSurfaceFrom()`, `SDL_FreeSurface()`, `Load(SDL_Surface&)`
- Notes: Creates surface with appropriate RGBA masks based on platform endianness; temporary surface freed after load

### Load (resource ID variant)
- Signature: `bool Load(int picture_resource)`
- Purpose: Load image from game resource by ID
- Inputs: `picture_resource` ΓÇô resource identifier
- Outputs: `true` if successful, `false` otherwise
- Side effects: Loads resource from images manager, converts to SDL_Surface via `picture_to_surface()`
- Calls: `get_picture_resource_from_images()`, `picture_to_surface()`, `Load(SDL_Surface&)`
- Notes: Chains through external resource system; resource cleanup delegated to helper functions

### Load (SDL_Surface variants)
- Signature: `bool Load(const SDL_Surface& s)` and `bool Load(const SDL_Surface& s, const SDL_Rect& src)`
- Purpose: Load image from existing SDL_Surface, optionally with source region
- Inputs: `s` ΓÇô source surface; `src` ΓÇô source rectangle (optional, defaults to full surface)
- Outputs: `true` if successful, `false` otherwise
- Side effects: Calls `Unload()` first; creates new RGB surface with copy blend mode disabled; copies pixels via `SDL_BlitSurface()`; initializes scaled dimensions and crop rectangle
- Calls: `Unload()`, `SDL_CreateRGBSurface()`, `SDL_SetSurfaceBlendMode()`, `SDL_BlitSurface()`
- Notes: Sets blend mode to `SDL_BLENDMODE_NONE` to ensure alpha is copied, not blended; crop_rect initialized to full source dimensions

### Unload
- Signature: `void Unload()`
- Purpose: Free all allocated SDL surfaces and reset dimensions
- Inputs: None
- Outputs: None
- Side effects: Frees `m_surface`, `m_disp_surface`, `m_scaled_surface`; zeroes width/height of both src rects
- Calls: `SDL_FreeSurface()` (3├ù)
- Notes: Safe to call multiple times (null-checks implicit in SDL_FreeSurface)

### Rescale
- Signature: `void Rescale(float width, float height)`
- Purpose: Change scaled dimensions and proportionally adjust crop rectangle
- Inputs: `width`, `height` ΓÇô new scaled dimensions
- Outputs: None
- Side effects: Updates `m_scaled_src.w`, `m_scaled_src.h`; proportionally scales `crop_rect` based on dimension ratio
- Calls: None
- Notes: Only rescales dimensions that actually change; crop adjustments preserve relative position and size

### Draw
- Signature: `void Draw(SDL_Surface *dst_surface, const Image_Rect& dst, const Image_Rect& src)`
- Purpose: Blit image to destination surface with optional rescaling
- Inputs: `dst_surface` ΓÇô target SDL_Surface; `dst` ΓÇô destination rectangle; `src` ΓÇô source cropping rectangle
- Outputs: None
- Side effects: Converts `m_surface` to `m_disp_surface` (BGRA8888) on first call; creates/updates `m_scaled_surface` if scaled dimensions differ from original; calls `SDL_BlitSurface()` to copy pixels
- Calls: `Loaded()`, `SDL_ConvertSurfaceFormat()`, `rescale_surface()`, `SDL_FreeSurface()`, `SDL_BlitSurface()`
- Notes: Lazily creates display and scaled surfaces; early-returns if image not loaded or surfaces unavailable; caches scaled surface (invalidates only if dimensions change)

### Dimension Queries
- Functions: `Width()`, `Height()`, `UnscaledWidth()`, `UnscaledHeight()`
- Purpose: Return current or original image dimensions
- Notes: Width/Height return scaled dimensions; Unscaled variants return original `m_src` dimensions

### Destructor
- Signature: `~Image_Blitter()`
- Purpose: Clean up allocated resources
- Inputs: None
- Outputs: None
- Side effects: Calls `Unload()`
- Calls: `Unload()`

## Control Flow Notes
**Lifecycle**: Constructor ΓåÆ (Load) ΓåÆ (Rescale/Draw)* ΓåÆ Unload/Destructor

- **Initialization**: Constructor zeroes all surfaces and rectangles
- **Loading**: One `Load()` variant populates `m_surface` and initializes `m_src`, `m_scaled_src`, `crop_rect`
- **Rendering**: `Draw()` lazily converts and rescales on-demand; cached surfaces reused until dimensions change
- **Cleanup**: `Unload()` explicitly frees surfaces; destructor ensures cleanup

## External Dependencies
- **SDL2**: `SDL_Surface`, `SDL_Rect`, `SDL_CreateRGBSurfaceFrom()`, `SDL_CreateRGBSurface()`, `SDL_FreeSurface()`, `SDL_SetSurfaceBlendMode()`, `SDL_BlitSurface()`, `SDL_ConvertSurfaceFormat()`
- **ImageLoader** (header): `ImageDescriptor`
- **images.h**: `get_picture_resource_from_images()`, `picture_to_surface()`, `rescale_surface()`
- **Platform detection**: `PlatformIsLittleEndian()`
- **cseries.h** (via Image_Blitter.h): Type definitions (`uint32`)
