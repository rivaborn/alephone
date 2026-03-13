# Source_Files/RenderMain/ImageLoader_Shared.cpp

## File Purpose
Implements image loading and decompression for DDS (DirectDraw Surface) files with support for DXTC texture compression formats (DXTC1/3/5). Provides mipmap management, format conversion, and DXTC decompression adapted from DevIL.

## Core Responsibilities
- Calculate and navigate mipmap chains (size, pointers, hierarchical access)
- Load and parse DDS file headers with endianness handling
- Load individual mipmaps with format-specific handling (RGBA8, DXTC1/3/5)
- Decompress DXTC-compressed textures to RGBA8
- Resize/minify images and convert between formats
- Premultiply alpha channel for blending

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Color8888` | struct | 32-bit RGBA color with per-byte channel access |
| `Color565` | struct | 16-bit RGB color with 5:6:5 bit-field packing |
| `DXTColBlock` | struct | DXT color block: 2 color endpoints + 4x4 palette indices |
| `DXTAlphaBlockExplicit` | struct | DXTC3 alpha block: 16 explicit 4-bit alpha values |
| `DXTAlphaBlock3BitLinear` | struct | DXTC5 alpha block: 2 endpoints + 48-bit interpolation indices |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `padfour` | inline function | static | Rounds integer up to next multiple of 4 |

## Key Functions / Methods

### GetMipMapSize
- Signature: `int GetMipMapSize(int level) const`
- Purpose: Calculate byte size of a mipmap level accounting for format-specific block packing
- Inputs: `level` ΓÇô mipmap index (0 = full resolution)
- Outputs/Return: Size in bytes; 0 if level invalid
- Side effects: Prints diagnostic to stderr on invalid level
- Notes: DXTC formats pack 4├ù4 pixel blocks; levels shift dimensions right by `level` bits with minimum 1 pixel per dimension

### GetMipMapPtr
- Signature: `const uint32 *GetMipMapPtr(int Level) const` and `uint32 *GetMipMapPtr(int Level)`
- Purpose: Return pointer to mipmap level data within pixel buffer
- Inputs: `Level` ΓÇô mipmap index
- Outputs/Return: Pointer to level data, or NULL if out of bounds
- Side effects: None
- Calls: `GetMipMapSize` (cumulative sum to find offset)
- Notes: Sums all prior mipmap sizes; pointer arithmetic assumes uint32 alignment

### Resize
- Signature: `void Resize(int _Width, int _Height)` / `void Resize(int _Width, int _Height, int _TotalBytes)`
- Purpose: Reallocate and reinitialize pixel buffer
- Inputs: Dimensions and optional total size
- Outputs/Return: None
- Side effects: Deletes old buffer, allocates new one; loses existing image data
- Notes: Size is always `_Width * _Height * 4` in first version (RGBA8 assumed)

### Minify
- Signature: `bool Minify()`
- Purpose: Reduce image to next lower mipmap level or halve RGBA8 dimensions
- Inputs: None
- Outputs/Return: true if successful, false if already 1├ù1 or GL unavailable
- Side effects: Modifies Width, Height, Size, MipMapCount, Pixels
- Calls: `GetMipMapPtr`, `GetMipMapSize`, `OGL_IsActive`, `gluScaleImage` (GL)
- Notes: Requires OpenGL active for RGBA8 format; DXTC formats must have mipmaps pre-built

### LoadDDSFromFile
- Signature: `bool LoadDDSFromFile(FileSpecifier& File, int flags, int actual_width, int actual_height, int maxSize)`
- Purpose: Parse DDS header and load mipmap data with format/resolution negotiation
- Inputs: File, load flags (mipmaps, DXTC support, resize), original/max dimensions
- Outputs/Return: true on success, false on parse/format error
- Side effects: Allocates buffer, modifies all ImageDescriptor members
- Calls: `LoadMipMapFromFile`, `SkipMipMapFromFile`, `Minify`, `MakeRGBA`, `MakeDXTC3`, SDL stream parsing
- Notes: Rejects cubemaps/volumes; recalculates mipmap count for power-of-two resizing; validates mipmap chain completeness with 1-level tolerance (XBLA workaround)

### LoadMipMapFromFile
- Signature: `bool LoadMipMapFromFile(OpenedFile& file, int flags, int level, DDSURFACEDESC2 &ddsd, int skip)`
- Purpose: Decode and load a single mipmap level from file into buffer
- Inputs: File, flags, mipmap level, DDS header, skip offset
- Outputs/Return: true if read successful
- Side effects: Writes decompressed pixels to buffer at calculated offset
- Calls: `GetBuffer`, `GetMipMapSize`, `SDL_CreateRGBSurfaceFrom`, `SDL_BlitSurface`, `SDL_FreeSurface`, `memset`, `file.Read`
- Notes: RGBA8 uses SDL for arbitrary bit-depth conversion; DXTC formats copy blocks with padding; handles endianness via `PlatformIsLittleEndian()`

### SkipMipMapFromFile
- Signature: `bool SkipMipMapFromFile(OpenedFile& File, int flags, int level, DDSURFACEDESC2 &ddsd)`
- Purpose: Advance file position past mipmap without loading
- Inputs: File, flags, level, DDS header
- Outputs/Return: true if position updated
- Side effects: Changes file position
- Calls: `File.GetPosition`, `File.SetPosition`
- Notes: Used to skip over higher-resolution levels when maxSize limit applied

### MakeRGBA
- Signature: `bool MakeRGBA()`
- Purpose: Decompress DXTC format to RGBA8, preserving all mipmap levels
- Inputs: None
- Outputs/Return: true if decompression successful
- Side effects: Replaces Pixels buffer, updates Format and Size
- Calls: `GetMipMapSize`, `GetMipMapPtr`, `DecompressDXTC1/3/5`
- Notes: Allocates new buffer; swaps old buffer with decompressed result; caller must free old data

### PremultiplyAlpha
- Signature: `void PremultiplyAlpha()`
- Purpose: Multiply RGB by alpha channel for proper blending
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies Pixels in-place; sets PremultipliedAlpha flag
- Notes: Skips fully opaque (alpha=255) and fully transparent (alpha=0) pixels; uses byte pointer arithmetic for endianness-aware channel access

### DecompressDXTC1 / DecompressDXTC3 / DecompressDXTC5
- Signature: `static bool DecompressDXTC[1/3/5](uint32 *out, int width, int height, uint32 *in)`
- Purpose: Decode DXTC block-compressed data to RGBA8
- Inputs: Output buffer, dimensions, compressed input
- Outputs/Return: true (always succeeds if called correctly)
- Side effects: Fills output buffer with RGBA pixels
- Calls: `SDL_SwapLE16`, `SDL_SwapLE32` (endianness), `PlatformIsLittleEndian`
- Notes: 
  - DXTC1: 2 color endpoints, interpolate 2 additional; 1 bit per alpha (transparent/opaque)
  - DXTC3: 2 color endpoints + explicit 4-bit alpha per pixel
  - DXTC5: 2 color endpoints + interpolated 3-bit alpha (6 or 8 alpha values depending on endpoint order)
  - All process 4├ù4 pixel blocks; boundary checking prevents writes beyond image dimensions

## Control Flow Notes
**Initialization:**  
`LoadDDSFromFile` entry point parses header, determines format (RGBA8/DXTC1/3/5), and delegates to `LoadMipMapFromFile` or `SkipMipMapFromFile` per mipmap.

**Format conversion (post-load):**  
If flags forbid DXTC (e.g., old hardware), `MakeRGBA` decompresses all levels; if DXTC1 with alpha flag, `MakeDXTC3` doubles size by prepending alpha blocks.

**Mipmap scaling:**  
If OpenGL unavailable or file lacks mipmaps, `Minify` reduces resolution iteratively until under `maxSize` threshold.

**Alpha premultiplication:**  
Applied last if requested; operates only on RGBA8 format.

## External Dependencies
- **AStream.h**: `AIStreamLE` for binary parsing with endianness
- **SDL2**: `SDL_CreateRGBSurfaceFrom`, `SDL_BlitSurface`, `SDL_FreeSurface`, `SDL_SetSurfaceBlendMode`, `SDL_SwapLE16/32` (endian swaps)
- **OpenGL** (conditional): `gluScaleImage` for RGBA8 downscaling; `OGL_IsActive()` gate
- **DDS.h**: `DDSURFACEDESC2` structure and DDSD/DDPF/DDSCAPS flag constants
- **ImageLoader.h**: `ImageDescriptor` class declaration, format enums
- **cstypes.h**: Fixed-size integer types (`uint32`, `uint8`, etc.), `FOUR_CHARS_TO_INT` macro
- **Logging.h**: `logWarning()` macro
- **Undefined (defined elsewhere)**: `PlatformIsLittleEndian()`, `NextPowerOfTwo()`, `OpenedFile` (file abstraction), `FileSpecifier`
