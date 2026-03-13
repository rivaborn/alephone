# Source_Files/RenderOther/Shape_Blitter.h

## File Purpose
Defines the `Shape_Blitter` class for rendering 2D bitmaps from the game's Shapes file format to screen surfaces. Supports both OpenGL and SDL rendering backends with visual effects (tinting, rotation, cropping) for UI and sprite rendering.

## Core Responsibilities
- Load shape bitmaps from Shapes file collections with support for texture categories (walls, landscapes, sprites, weapons, interface)
- Manage SDL surface buffers (original and scaled copies)
- Perform dimension queries (scaled and unscaled)
- Render shapes via OGL_Draw() or SDL_Draw() with optional tinting and rotation
- Apply source-region cropping via crop_rect before rendering
- Clean up allocated SDL resources on destruction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Shape_Texture_* enum (Wall, Landscape, Sprite, WeaponInHand, Interface) | enum constants | Categorize shape types for different in-game asset contexts |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor
- **Signature:** `Shape_Blitter(short collection, short frame_index, short texture_type, short clut_index = 0)`
- **Purpose:** Load and initialize a shape from the Shapes file collection
- **Inputs:** collection index, frame index within collection, texture type category, optional color lookup table index
- **Outputs/Return:** Initialized object with loaded shape data
- **Side effects:** Allocates SDL surface(s) and loads bitmap data from Shapes file
- **Calls:** (details in .cpp)
- **Notes:** No visible input validation; assumes valid indices provided by caller

### Rescale
- **Signature:** `void Rescale(float width, float height)`
- **Purpose:** Resize the loaded shape bitmap to specified dimensions
- **Inputs:** Target width and height in pixels
- **Outputs/Return:** None (modifies m_scaled_surface cache)
- **Side effects:** Recreates scaled SDL surface; invalidates previous scaling
- **Calls:** SDL scaling functions (in .cpp)
- **Notes:** Called before Draw() for custom scaling; supports fractional dimensions

### OGL_Draw
- **Signature:** `void OGL_Draw(const Image_Rect& dst)`
- **Purpose:** Render shape to OpenGL context at destination rectangle
- **Inputs:** dst ΓÇö destination rect (position and size)
- **Outputs/Return:** None
- **Side effects:** I/O to active OpenGL context; applies tint_color and rotation
- **Calls:** OpenGL rendering functions (in .cpp)
- **Notes:** Behavior depends on active GL context; respects crop_rect and visual effect parameters

### SDL_Draw
- **Signature:** `void SDL_Draw(SDL_Surface *dst_surface, const Image_Rect& dst)`
- **Purpose:** Render shape to SDL surface using software blitting
- **Inputs:** target SDL surface, destination rect
- **Outputs/Return:** None
- **Side effects:** Modifies pixels in dst_surface
- **Calls:** SDL blit/transform functions (in .cpp)
- **Notes:** Software rendering path; applies tint, rotation, and crop_rect if supported

### Destructor
- **Signature:** `~Shape_Blitter()`
- **Purpose:** Release allocated SDL surfaces
- **Outputs/Return:** None
- **Side effects:** Frees m_surface and m_scaled_surface
- **Notes:** Essential for memory cleanup; no move semantics defined

### Notes
- Dimension query methods `Width()`, `Height()`, `UnscaledWidth()`, `UnscaledHeight()` retrieve scaled and original bitmap dimensions.

## Public Data Members (Visual Effects)
- `tint_color_r/g/b/a` (float): RGBA tint multiplier; (1,1,1,1) is untinted
- `rotation` (float): Clockwise rotation angle in degrees around destination rect center
- `crop_rect` (Image_Rect): Source region clipping rectangle (default: full bitmap)

## Control Flow Notes
- Shape loading occurs at construction; bitmap immutable afterward
- Optional visual parameters (tint, rotation, crop) set via public members before rendering
- Rendering dispatches via OGL_Draw() or SDL_Draw(); caller chooses backend
- Standalone utility; not integrated into main game loop (supports 2D UI and sprite rendering)

## External Dependencies
- `cseries.h` ΓÇö platform abstraction, SDL2 includes
- `Image_Blitter.h` ΓÇö parent/related class defining `Image_Rect` struct and SDL patterns
- `map.h` ΓÇö Shapes file format context
- SDL2 ΓÇö surface and rendering
- `<vector>`, `<set>` ΓÇö standard containers (included but usage likely in .cpp)
