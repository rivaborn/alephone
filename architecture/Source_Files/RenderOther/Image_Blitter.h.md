# Source_Files/RenderOther/Image_Blitter.h

## File Purpose
Defines the `Image_Blitter` class for rendering 2D UI images to SDL surfaces with support for scaling, cropping, tinting, and rotation. Part of the Aleph One game engine's 2D rendering system.

## Core Responsibilities
- Load images from `ImageDescriptor`, resource IDs, or SDL surfaces with optional source cropping
- Manage scaled and original image buffers
- Render images to destination surfaces with transformations
- Apply per-image tinting and rotation effects
- Expose image dimensions (scaled and unscaled)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Image_Rect` | struct | Float-based rectangle (x, y, width, height) for positioning and sizing; implicitly convertible from `SDL_Rect` |
| `Image_Blitter` | class | Encapsulates image loading, scaling, and blitting operations |

## Global / File-Static State
None.

## Key Functions / Methods

### Load
- **Signature:** `bool Load(const ImageDescriptor& image); bool Load(int picture_resource); bool Load(const SDL_Surface& s); bool Load(const SDL_Surface& s, const SDL_Rect& src);`
- **Purpose:** Load image data from various sources (descriptor, resource ID, or SDL surface); optionally specify source cropping region.
- **Inputs:** Image source (descriptor/resource ID/surface) and optional source rectangle for cropping.
- **Outputs/Return:** Boolean indicating load success.
- **Side effects:** Allocates/replaces internal surface pointers (`m_surface`, `m_disp_surface`, `m_scaled_surface`).
- **Calls:** Defined elsewhere (implementation not in header).

### Rescale
- **Signature:** `void Rescale(float width, float height);`
- **Purpose:** Rescale the loaded image to specified dimensions.
- **Inputs:** Target width and height.
- **Outputs/Return:** None.
- **Side effects:** Updates `m_scaled_surface` and `m_scaled_src`.

### Draw
- **Signature:** `virtual void Draw(SDL_Surface *dst_surface, const Image_Rect& dst); virtual void Draw(SDL_Surface *dst_surface, const Image_Rect& dst, const Image_Rect& src);`
- **Purpose:** Render image to destination surface with optional custom source rectangle (default uses `crop_rect`).
- **Inputs:** Destination SDL surface, destination rect, and optional source rect.
- **Outputs/Return:** None.
- **Side effects:** Modifies destination surface pixel data; respects `tint_color_*` and `rotation` members.
- **Calls:** Second overload called by first.
- **Notes:** Tinting is applied as (r, g, b, a) multipliers; rotation is in degrees clockwise about destination rect center.

### Loaded
- **Signature:** `bool Loaded();`
- **Purpose:** Check if an image is currently loaded.
- **Inputs:** None.
- **Outputs/Return:** Boolean.

### Dimension Accessors
- **Signature:** `float Width(); float Height(); int UnscaledWidth(); int UnscaledHeight();`
- **Purpose:** Retrieve scaled or original image dimensions.
- **Outputs/Return:** Dimensions (float for scaled, int for unscaled).

## Control Flow Notes
Not inferable from header. Likely integrated into a frame-based 2D UI renderer: Load images once, apply tinting/rotation/scaling as needed per-frame, then Draw during the UI render pass.

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_Rect` (graphics primitives)
- **ImageLoader.h:** `ImageDescriptor` (cross-format image container)
- **cseries.h:** Standard engine utilities and includes
- **STL:** `<vector>`, `<set>` (declared but not used in header)
