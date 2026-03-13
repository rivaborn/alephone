# Source_Files/RenderMain/ImageLoader_SDL.cpp

## File Purpose
SDL-based implementation for loading image files into `ImageDescriptor` objects. Converts image data from files (DDS, PNG, BMP, etc.) to 32-bit RGBA surfaces and optionally extracts opacity/grayscale information.

## Core Responsibilities
- Load image files from disk via SDL_image or SDL BMP fallback
- Convert loaded images to platform-correct 32-bit RGBA format (handling endianness)
- Resize images to powers of two if requested
- Process two image modes: color data and opacity/grayscale data
- Extract opacity from grayscale/alpha channels and store as per-pixel alpha
- Delegate DDS format loading to `LoadDDSFromFile()`
- Validate dimension/scale consistency when loading opacity over existing color data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ImageDescriptor` | class | Container for loaded image data, dimensions, scaling, and format metadata |
| `FileSpecifier` | class | File path abstraction; defined elsewhere |
| `OpenedFile` | class | File handle wrapper around SDL_RWops; defined elsewhere |
| `SDL_Surface` | struct | SDL's in-memory pixel surface representation |
| `SDL_RWops` | struct | SDL's I/O abstraction for file reading |

## Global / File-Static State
None.

## Key Functions / Methods

### ImageDescriptor::LoadFromFile
- **Signature:** `bool LoadFromFile(FileSpecifier& File, int ImgMode, int flags, int actual_width, int actual_height, int maxSize)`
- **Purpose:** Load an image file and populate the `ImageDescriptor` with pixel data; supports color or opacity modes.
- **Inputs:**
  - `File`: FileSpecifier pointing to the image file
  - `ImgMode`: `ImageLoader_Colors` or `ImageLoader_Opacity` (defines whether to load RGB or extract opacity)
  - `flags`: Bitfield (`ImageLoader_ResizeToPowersOfTwo`, `ImageLoader_ImageIsAlreadyPremultiplied`, etc.)
  - `actual_width`, `actual_height`: Optional original dimensions for UV scaling
  - `maxSize`: Size limit (used by DDS loader)
- **Outputs/Return:** `true` on success, `false` on failure (file not found, invalid format, size mismatch, etc.)
- **Side effects:**
  - Allocates/resizes `this->Pixels` buffer via `Resize()`
  - Sets `this->Width`, `this->Height`, `this->VScale`, `this->UScale`, `this->MipMapCount`
  - Sets `this->PremultipliedAlpha` if flag set
  - Opens file, reads data, closes via RAII (`OpenedFile`)
  - Allocates and frees temporary SDL surfaces
- **Calls:**
  - `LoadDDSFromFile()` (if ImgMode is Colors)
  - `File.Open()`, `OpenedFile::GetRWops()`
  - `IMG_Load_RW()` or `SDL_LoadBMP_RW()` depending on `HAVE_SDL_IMAGE`
  - `NextPowerOfTwo()` (defined elsewhere)
  - `PlatformIsLittleEndian()` (defined elsewhere)
  - `Resize()` on this descriptor
  - `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`, `SDL_SetSurfaceBlendMode()`
  - `memcpy()` or per-pixel opacity extraction loop
  - `PIN()` macro (clamping, defined elsewhere)
- **Notes:**
  - If `ImgMode == ImageLoader_Opacity`, assumes color data already loaded; validates matching dimensions and scales
  - Opacity mode converts RGB to grayscale via simple average, stores in alpha channel of existing pixel buffer
  - Disables SDL_SRCALPHA blending before blit to avoid unwanted alpha premultiplication during copy
  - Uses platform-aware RGBA32 byte order (little-endian vs. big-endian)
  - Returns early if DDS succeeds (DDS is preferred for color mode)

## Control Flow Notes
This is an initialization/load-time function, typically called during asset loading (before rendering). Called in sequence: first for color (`ImageLoader_Colors`), then optionally for opacity (`ImageLoader_Opacity`) to overlay alpha on the same image descriptor. Not part of frame/render loop.

## External Dependencies
- **Includes:** `ImageLoader.h`, `FileHandler.h`, `<SDL2/SDL_image.h>` (conditional), `<cmath>`
- **Symbols defined elsewhere:**
  - `ImageDescriptor::LoadDDSFromFile()`, `ImageDescriptor::Resize()`, `ImageDescriptor::GetPixelBasePtr()` (class methods)
  - `FileSpecifier::Open()` (file abstraction)
  - `OpenedFile::GetRWops()` (SDL RWops accessor)
  - `NextPowerOfTwo()`, `PlatformIsLittleEndian()`, `PIN()` (utility functions/macros)
  - `IMG_Load_RW()`, `SDL_LoadBMP_RW()`, `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`, `SDL_SetSurfaceBlendMode()` (SDL2 API)
  - `temporary`, `csprintf()`, `vassert()` (logging/assertion, defined elsewhere)
