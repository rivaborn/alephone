# Source_Files/RenderMain/OGL_Textures.cpp

## File Purpose
Implements OpenGL texture management for Aleph One, handling texture loading, caching, VRAM management, and special effects (infravision, silhouettes, glow/bump mapping) for walls, landscapes, sprites, and UI elements.

## Core Responsibilities
- Manage texture state, allocation, and GPU memory lifecycle
- Load textures from Marathon bitmap data with format conversion
- Implement frame-tick-based texture purging to conserve VRAM
- Support substitute textures and texture replacement/patching
- Apply visual effects: infravision tinting, silhouettes, glow mapping
- Configure texture wrapping, filtering, and matrix transformations for different texture types
- Handle multiple texture formats: RGBA8, DXTC1/3/5, indexed color
- Track texture statistics and usage patterns

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `TextureState` | struct | Manages single texture's GL ID, usage count, generation state; embedded in texture sets |
| `CollBitmapTextureState` | struct | Groups texture states (normal/glow/bump) per bitmap; indexed by color table |
| `TxtrTypeInfoData` | struct | Filter/format configuration per texture type (walls, landscapes, sprites, etc.) |
| `InfravisionData` | struct | Tint RGB values + enabled flag per collection for infravision effect |
| `OGL_TexturesStats` | struct | Global counters: in-use textures, total age, bind count, min/max/long-setup times |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gGLTxStats` | `OGL_TexturesStats` | global | Texture performance metrics |
| `TxtrTypeInfoList[]` | `TxtrTypeInfoData[7]` | static | Per-type filter and color format config |
| `ModelSkinInfo` | `TxtrTypeInfoData` | static | Configuration for model skin textures |
| `useSGISMipmaps` | bool | static | Whether GPU-side mipmap generation available |
| `IVDataList[]` | `InfravisionData[32]` | static | Infravision tint for each collection |
| `InfravisionActive` | bool | static | Global infravision enable flag |
| `sgActiveTextureStates` | `std::list<TextureState*>` | static | Active textures for per-frame cleanup |
| `TextureStateSets[][]` | `CollBitmapTextureState*[7][32]` | static | 2D texture state array by type and collection |
| `flatBumpTextureID` | `GLuint` | static | Flat normal map texture (0x80, 0x80, 0xFF, 0x80) for default bump mapping |

## Key Functions / Methods

### TextureState::Allocate
- **Signature:** `bool Allocate(short txType)`
- **Purpose:** Allocate OpenGL texture IDs for this state object
- **Inputs:** `txType` ΓÇô texture type enum (wall, landscape, sprite, etc.)
- **Outputs/Return:** `true` if newly allocated, `false` if already in use
- **Side effects:** Calls `glGenTextures()`, updates `gGLTxStats.inUse` and active list
- **Calls:** `glGenTextures()`, `sgActiveTextureStates.push_front()`
- **Notes:** Safe to call multiple times; idempotent after first call

### TextureState::Use
- **Signature:** `bool Use(int Which)`
- **Purpose:** Bind texture and indicate whether loading is needed
- **Inputs:** `Which` ΓÇô texture index (Normal=0, Glowing=1, Bump=2)
- **Outputs/Return:** `true` if texture not yet generated, `false` if already loaded
- **Side effects:** Calls `glBindTexture()`, increments `IDUsage[Which]`, sets `TexGened[Which]`
- **Calls:** `glBindTexture(GL_TEXTURE_2D, IDs[Which])`

### TextureState::Reset
- **Signature:** `void Reset()`
- **Purpose:** Release OpenGL texture and reset state
- **Side effects:** Calls `glDeleteTextures()`, removes from active list, decrements stats
- **Calls:** `glDeleteTextures()`, `sgActiveTextureStates.remove()`

### TextureState::FrameTick
- **Signature:** `void FrameTick()`
- **Purpose:** Per-frame update; release textures unused for N frames based on type
- **Side effects:** Increments `unusedFrames`, calls `Reset()` when threshold exceeded
- **Thresholds:** Walls 300 frames (10s), sprites 450 (15s), weapons/HUD 600 (20s), landscapes never
- **Notes:** Resets usage counters if texture was used this frame

### OGL_StartTextures
- **Signature:** `void OGL_StartTextures()`
- **Purpose:** Initialize texture system at startup
- **Side effects:** Allocates `TextureStateSets[][]` arrays, configures filter/format from preferences, detects SGIS_generate_mipmap
- **Calls:** `new`, `Get_OGL_ConfigureData()`, `OGL_CheckExtension()`

### OGL_StopTextures
- **Signature:** `void OGL_StopTextures()`
- **Purpose:** Shut down texture system; deallocate all resources
- **Side effects:** Deletes `TextureStateSets` arrays, calls `OGL_Blitter::StopTextures()`, `FontSpecifier::OGL_ResetFonts()`, `glDeleteTextures()`
- **Calls:** `delete[]`, `OGL_Blitter::StopTextures()`, `glDeleteTextures()`

### OGL_FrameTickTextures
- **Signature:** `void OGL_FrameTickTextures()`
- **Purpose:** Per-frame texture maintenance
- **Side effects:** Calls `FrameTick()` on all active textures
- **Calls:** `TextureState::FrameTick()` for each in `sgActiveTextureStates`

### TextureManager::Setup
- **Signature:** `bool Setup()`
- **Purpose:** Main texture initialization; parse shape descriptor, load or reuse texture
- **Inputs:** Member fields `ShapeDesc`, `TransferMode`, `TextureType`, `Landscape_AspRatExp`, etc.
- **Outputs/Return:** `true` on success, `false` if bitmap not found or texture setup failed
- **Side effects:** Populates `NormalImage`, `GlowImage`, `OffsetImage`; may allocate new `ImageDescriptor` objects; calls `FindColorTables()` or `LoadSubstituteTexture()`
- **Calls:** `GET_DESCRIPTOR_*()`, `ModifyCLUT()`, `LoadSubstituteTexture()`, `SetupTextureGeometry()`, `FindColorTables()`, `ImageDescriptor` ctor
- **Notes:** Complex; handles texture caching via `CTStates[CTable]`, sprite scale/offset persistence

### TextureManager::LoadSubstituteTexture
- **Signature:** `bool LoadSubstituteTexture()`
- **Purpose:** Load replacement texture if available; validate dimensions
- **Outputs/Return:** `true` if substitute loaded, `false` otherwise
- **Side effects:** Sets `TxtrWidth`, `TxtrHeight`, `TxtrOptsPtr->Substitution`; applies infravision/silhouette if needed
- **Calls:** `NormalImg.GetWidth/Height()`, `NextPowerOfTwo()`, `IsLandscapeFlatColored()`, `SetPixelOpacities()`, `FindSilhouetteVersion()`
- **Notes:** Enforces power-of-2 for tiling unless `npotTextures` enabled; handles M1 compatibility

### TextureManager::SetupTextureGeometry
- **Signature:** `bool SetupTextureGeometry()`
- **Purpose:** Calculate texture dimensions and offsets from bitmap; apply aspect ratio corrections
- **Outputs/Return:** `true` on success, `false` if power-of-2 requirement violated
- **Side effects:** Sets `TxtrWidth`, `TxtrHeight`, `WidthOffset`, `HeightOffset`, `U_Scale`, `U_Offset`, `V_Scale`, `V_Offset`
- **Calls:** `shapes_file_is_m1()`, `IsLandscapeFlatColored()`, `NextPowerOfTwo()`, `MAX()`
- **Notes:** Handles M1 truncation (max 128px), landscape aspect ratio, sprite border padding (2px for mipmaps)

### TextureManager::FindColorTables
- **Signature:** `void FindColorTables()`
- **Purpose:** Convert Marathon shading tables to OpenGL RGBA format; detect self-luminous colors for glow
- **Side effects:** Populates `NormalColorTable[]`, `GlowColorTable[]`; sets `IsGlowing` flag; modifies opacity
- **Calls:** `IsSilhouetteTable()`, `IsInfravisionTable()`, `FindOGLColorTable()`, `SetPixelOpacitiesRGBA()`, `MakeEightBit()`
- **Notes:** Halves opacity of glow color to simulate software rendering; silhouettes map to white

### TextureManager::PlaceTexture
- **Signature:** `void PlaceTexture(const ImageDescriptor *Image, bool normal_map = false)`
- **Purpose:** Upload texture data to GPU; configure filtering and wrapping
- **Inputs:** `Image` ΓÇô image data to upload; `normal_map` ΓÇô whether this is a bump map (affects sRGB)
- **Side effects:** Calls `glTexImage2D()` or `glCompressedTexImage2DARB()`, `glTexParameteri()`, `glTexEnvi()`
- **Calls:** `glTexImage2D()`, `gluBuild2DMipmaps()`, `glCompressedTexImage2DARB()`, `glTexParameteri()`, `glTexEnvi()`, `glGetFloatv()`
- **Notes:** Supports RGBA8, DXTC1/3/5; auto-converts to sRGB unless interface/weapons; uses GLU or SGIS mipmap generation; anisotropic filtering for walls

### TextureManager::RenderNormal / RenderGlowing / RenderBump
- **Signature:** `void RenderNormal()`, `void RenderGlowing()`, `void RenderBump()`
- **Purpose:** Bind and place texture for rendering; track statistics
- **Side effects:** Calls `TxtrStatePtr->Allocate()` and `Use*()`, then `PlaceTexture()`; updates `gGLTxStats`
- **Calls:** `TextureState::Allocate()`, `TextureState::Use*()`, `PlaceTexture()`

### TextureManager::SetupTextureMatrix / RestoreTextureMatrix
- **Signature:** `void SetupTextureMatrix()`, `void RestoreTextureMatrix()`
- **Purpose:** Configure GL texture matrix (rotation, scale, translate) for coordinate transformation
- **Side effects:** Calls `glMatrixMode()`, `glLoadIdentity()`, `glRotatef()`, `glScalef()`, `glTranslatef()`
- **Notes:** Substituted textures rotated 90┬░ and flipped for walls/sprites; landscapes scaled/translated for aspect ratio

### FindOGLColorTable
- **Signature:** `static void FindOGLColorTable(int NumSrcBytes, byte *OrigColorTable, uint32 *ColorTable)`
- **Purpose:** Convert Marathon color table (16-bit ARGB 5551 or 32-bit ARGB 8888) to OpenGL RGBA 8888
- **Inputs:** `NumSrcBytes` ΓÇô 2 or 4; `OrigColorTable` ΓÇô source color data
- **Outputs/Return:** `ColorTable[]` ΓÇô RGBA 8888 output (256 entries)
- **Calls:** `Convert_16to32()`, `PlatformIsLittleEndian()`

### SetPixelOpacities / SetPixelOpacitiesRGBA
- **Signature:** `void SetPixelOpacities(OGL_TextureOptions& Options, ImageDescriptorManager &imageManager)`, `void SetPixelOpacitiesRGBA(OGL_TextureOptions& Options, int NumPixels, uint32 *Pixels)`
- **Purpose:** Apply opacity scale/shift; optionally use color luminance (Tomb Raider hack)
- **Inputs:** `Options.OpacityType` (Avg, Max, or pre-existing), `OpacityScale`, `OpacityShift`
- **Side effects:** Modifies alpha channel of pixels in-place
- **Calls:** `MakeEightBit()`, `PIN()`, dispatches to DXTC3/5 handlers if needed
- **Notes:** Tomb Raider hack: opacity = (R+G+B)/3, scaled and shifted; supports DXTC decompression if needed

### FindInfravisionVersionRGBA / FindSilhouetteVersion*
- **Signature:** `void FindInfravisionVersionRGBA(short Collection, GLfloat *Color)`, `void FindSilhouetteVersion(ImageDescriptorManager &imageManager)`
- **Purpose:** Apply infravision tint or silhouette effect to texture
- **Side effects:** Modifies color values in-place
- **Calls:** `SDL_SwapLE16()`, `PlatformIsLittleEndian()` for silhouettes
- **Notes:** Silhouette: sets RGB to white, preserves alpha; differs per format (RGBA vs DXTC1/3/5)

### SetInfravisionTint
- **Signature:** `bool SetInfravisionTint(short Collection, bool IsTinted, float Red, float Green, float Blue)`
- **Purpose:** Configure infravision tint for a collection
- **Inputs:** `Collection` index, `IsTinted` enable flag, RGB values [0,1]
- **Outputs/Return:** `true`
- **Side effects:** Updates `IVDataList[Collection]`

### IsInfravisionActive
- **Signature:** `bool& IsInfravisionActive()`
- **Purpose:** Get reference to global infravision state
- **Outputs/Return:** Reference to `InfravisionActive` bool

### LoadModelSkin
- **Signature:** `void LoadModelSkin(ImageDescriptor& SkinImage, short Collection, short CLUT)`
- **Purpose:** Load a model skin texture (similar to TextureManager but standalone)
- **Side effects:** Calls `glTexImage2D()`, `glTexParameteri()`, `glTexEnvi()`
- **Calls:** `glGetIntegerv()`, `glTexImage2D()`, `gluBuild2DMipmaps()`, `glTexParameteri()`

## Notes

- Texture caching uses a 2D array indexed by type and collection, with per-bitmap color-table subindexes.
- Frame-tick purging is aggressive: sprites released after 15 seconds of disuse, walls after 10 seconds.
- Substitute textures bypass geometric calculations and use ImageDescriptor dimensions directly.
- Infravision and silhouette are mutually exclusive visual effects applied during texture setup.
- sRGB conversion is automatically enabled for color textures unless they are UI or weapon skins.
- Anisotropic filtering is enabled only for wall textures if configured.

## Control Flow Notes

**Init ΓåÆ Per-frame ΓåÆ Shutdown:**
- `OGL_StartTextures()` allocates state arrays
- Each frame: `OGL_FrameTickTextures()` purges old textures
- `TextureManager::Setup()` called per-rendered texture; loads if new, reuses if cached
- `TextureManager::Render*()` methods bind textures each frame
- `OGL_StopTextures()` deallocates everything on shutdown

## External Dependencies
- **OpenGL:** `glGenTextures`, `glBindTexture`, `glDeleteTextures`, `glTexImage2D`, `glTexParameteri`, `glTexEnvi`, `gluBuild2DMipmaps`, `glCompressedTexImage2DARB`, `glGetIntegerv`
- **SDL:** `SDL_SwapLE16`, `SDL_endian.h`
- **Marathon engine:** shape descriptors, collection system (`get_bitmap_index`, `get_collection_colors`), map data, preferences
- **Image management:** `ImageDescriptor`, `ImageDescriptorManager`, `ImageLoader` (implied via includes)
- **Engine infrastructure:** `OGL_Setup.h`, `OGL_Render.h`, `OGL_Blitter.h`, `OGL_Headers.h` (OpenGL abstraction)
