# Source_Files/RenderMain/ImageLoader.h

## File Purpose
Defines the `ImageDescriptor` class for holding pixel data and metadata, along with image loading utilities. Supports loading images from files (particularly DDS format), mipmap operations, and format conversions (RGBA8, DXTC compression). Part of the Aleph One game engine's rendering subsystem.

## Core Responsibilities
- **Image data container**: Holds pixel buffer, dimensions, scales, and format information
- **File loading**: Load images from DDS files with optional mipmap support
- **Mipmap management**: Generate, access, and query mipmap levels
- **Format conversion**: Convert between RGBA8 and DXTC compression formats
- **Alpha blending**: Premultiply alpha channel
- **Copy-on-edit pattern**: Template class for lazy-copy resource management
- **Pixel access**: Direct and bulk access to image pixels

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ImageDescriptor` | class | Holds pixel data, dimensions, scales, mipmaps, and format metadata |
| `copy_on_edit<T>` | class template | Implements copy-on-edit pattern: lazy-copies original on first edit, allows read-only or mutable access |
| `ImageDescriptorManager` | typedef | `copy_on_edit<ImageDescriptor>`; manages lifecycle of image descriptors |
| `ImageFormat` | enum | Pixel format types: RGBA8, DXTC1, DXTC3, DXTC5, Unknown |

## Global / File-Static State
None.

## Key Functions / Methods

### ImageDescriptor::LoadFromFile
- **Signature**: `bool LoadFromFile(FileSpecifier& File, int ImgMode, int flags, int actual_width = 0, int actual_height = 0, int maxSize = 0)`
- **Purpose**: Load image from file (DDS or other supported format)
- **Inputs**: File specifier, image mode (Colors or Opacity), flags (ResizeToPowersOfTwo, CanUseDXTC, LoadMipMaps, etc.), optional dimensions and max size
- **Outputs/Return**: `true` if successful
- **Side effects**: Allocates and sets `Pixels` buffer, updates `Width`, `Height`, `Size`, `Format`, `MipMapCount`
- **Calls**: `LoadDDSFromFile()`, `LoadMipMapFromFile()`, `SkipMipMapFromFile()`
- **Notes**: Handles both color and opacity image modes; respects maxSize constraint

### ImageDescriptor::Resize
- **Signature**: `void Resize(int _Width, int _Height)` and `void Resize(int _Width, int _Height, int _TotalBytes)`
- **Purpose**: Reallocate pixel buffer to new dimensions
- **Inputs**: New width, height, optionally total byte size for mipmap chain
- **Outputs/Return**: None
- **Side effects**: Deallocates old `Pixels`, allocates new buffer, updates dimensions and size
- **Calls**: None visible
- **Notes**: Used during mipmap generation and format conversion

### ImageDescriptor::Minify
- **Signature**: `bool Minify()`
- **Purpose**: Generate next mipmap level (half-resolution)
- **Inputs**: None
- **Outputs/Return**: `true` if successful
- **Side effects**: Extends `Pixels` buffer with downsampled level, updates `MipMapCount`
- **Calls**: Implicitly calls pixel downsampling logic
- **Notes**: Called repeatedly to build full mipmap chain

### ImageDescriptor::GetMipMapPtr (const and non-const)
- **Signature**: `uint32 *GetMipMapPtr(int Level)` / `const uint32 *GetMipMapPtr(int Level) const`
- **Purpose**: Get pixel buffer pointer for a specific mipmap level
- **Inputs**: Mipmap level index (0 = full resolution)
- **Outputs/Return**: Pointer to level's pixel data
- **Side effects**: None
- **Calls**: None visible
- **Notes**: Returns nullptr if level out of range or not present

### ImageDescriptor::MakeRGBA, MakeDXTC3
- **Signature**: `bool MakeRGBA()` / `bool MakeDXTC3()`
- **Purpose**: Convert pixel format
- **Inputs**: None
- **Outputs/Return**: `true` if successful
- **Side effects**: Reallocates and converts pixel data, updates `Format`
- **Calls**: Indirectly allocates via `Resize()`
- **Notes**: Lossy conversions (e.g., DXTC compression)

### ImageDescriptor::PremultiplyAlpha
- **Signature**: `void PremultiplyAlpha()`
- **Purpose**: Blend alpha channel into RGB channels (pre-multiplication for correct blending)
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Modifies pixel data, sets `PremultipliedAlpha = true`
- **Calls**: None visible
- **Notes**: Affects all mipmap levels; idempotent after first call

### copy_on_edit<T>::edit
- **Signature**: `T* edit()` / `T* edit(T* copy)`
- **Purpose**: Get editable copy of resource; overloaded version takes ownership of external copy
- **Inputs**: Optional external copy (second form)
- **Outputs/Return**: Mutable pointer to copy
- **Side effects**: Lazy-allocates deep copy of original on first `edit()` call (if original exists); second form replaces internal copy
- **Calls**: None visible (allocates with `new`)
- **Notes**: Original remains unmodified; subsequent `get()` returns the copy

## Control Flow Notes
This header defines interfaces for image loading and manipulation, likely used during rendering setup or asset loading phases. `ImageDescriptor` objects are typically created, loaded from disk, optionally modified (format conversion, alpha premultiplication), then passed to renderer. The `copy_on_edit` pattern allows sharing image descriptors while safely allowing modifications without affecting originals.

## External Dependencies
- **DDS.h**: Direct Draw Surface format structures (`DDSURFACEDESC2`)
- **FileHandler.h**: File abstraction (`FileSpecifier`, `OpenedFile`)
- **cseries.h**: Core utilities and macros
- **\<vector>**: Standard library
- **cstypes.h** (via cseries.h): Fixed-width integer types (`uint32`, `int16`)

## Notes
- Constructor initializes `Pixels = NULL`, `Size = 0` (empty descriptor is valid)
- Pixel layout is row-major: `Pixels[Width * Vert + Horiz]`
- Public member `PremultipliedAlpha` breaks encapsulation (noted: "public so find silhouette version can unset"); likely a technical debt
- Trivial helpers summarized: `Clear()` deallocates buffer; `IsPresent()`, `IsPremultiplied()`, `GetPixel()`, scale getters are inline convenience methods
