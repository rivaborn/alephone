# Source_Files/RenderMain/OGL_Textures.h

## File Purpose
OpenGL texture manager header for Aleph One (Marathon-compatible engine). Defines structures and classes for managing texture lifecycle, state, rendering, and color transformation. Supports substitute textures, glow mapping, bump mapping, and infravision effects.

## Core Responsibilities
- Define texture configuration structures (TxtrTypeInfoData) with filter, resolution, and format parameters
- Manage texture state per collection bitmap (TextureState) with allocation, usage tracking, and per-frame updates
- Implement TextureManager class for loading, caching, and rendering textures with color tables
- Support texture coordinate scaling and offset for sprites and landscape geometry
- Handle substitute texture loading and fallback mechanisms
- Provide color format conversion (16-bit ARGB 1555 to 32-bit RGBA 8888)
- Manage infravision tinting and silhouette color transformations
- Track texture usage statistics and optimize per-frame housekeeping

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| TxtrTypeInfoData | struct | OpenGL texture format parameters: filter type, resolution level, color format |
| TextureState | struct | Manages OpenGL texture IDs (normal, glowing, bump), usage flags, scale/offset, and lifetime |
| CollBitmapTextureState | struct | Container for all TextureState objects for a collection bitmap |
| TextureManager | class | Main texture manager: loads, caches, and renders textures with color tables and geometry |
| OGL_TexturesStats | struct | Statistics: texture usage counts, bind operations, and setup timings |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| TxtrTypeInfoList[] | TxtrTypeInfoData[] | extern | Array of texture format configurations indexed by type |
| gGLTxStats | OGL_TexturesStats | extern | Aggregated texture operation statistics for profiling |

## Key Functions / Methods

### OGL_StartTextures()
- Purpose: Initialize texture accounting system
- Inputs: None
- Outputs/Return: void
- Side effects: Allocates texture state structures and initializes resource pools

### OGL_StopTextures()
- Purpose: Finalize and release all texture resources
- Inputs: None
- Outputs/Return: void
- Side effects: Deallocates textures and clears state

### OGL_FrameTickTextures()
- Purpose: Per-frame housekeeping: update texture usage counters, evict unused textures
- Inputs: None
- Outputs/Return: void
- Side effects: Updates all TextureState::unusedFrames counters, triggers garbage collection

### ModifyCLUT(short TransferMode, short CLUT)
- Purpose: Modify color-table index for special modes (infravision, silhouette)
- Inputs: TransferMode (rendering mode), CLUT (color lookup table index)
- Outputs/Return: short (adjusted CLUT index)

### TextureState::Allocate(short txType)
- Purpose: Allocate OpenGL texture IDs for normal/glow/bump maps
- Inputs: txType (texture classification)
- Outputs/Return: bool (allocation success)
- Side effects: Generates new OpenGL texture IDs if not already allocated

### TextureState::Use(int Which)
- Purpose: Mark a texture variant (Normal/Glowing/Bump) as in-use for this frame
- Inputs: Which (texture variant index)
- Outputs/Return: bool (texture is valid)
- Side effects: Resets unusedFrames counter

### TextureManager::Setup()
- Purpose: Initialize texture manager with shape descriptor, color tables, and geometry
- Inputs: ShapeDesc, Texture, ShadingTables, TransferMode, LandscapeVertRepeat (public members)
- Outputs/Return: bool (setup success)
- Side effects: Loads substitute textures, calculates dimensions, allocates buffers, fills color tables

### TextureManager::RenderNormal() / RenderGlowing() / RenderBump()
- Purpose: Bind and configure texture variant for rendering
- Inputs: None (uses internal state)
- Outputs/Return: void
- Side effects: Sets OpenGL texture state, calls glBindTexture, updates texture matrix

### Convert_16to32(uint16 InPxl)
- Signature: inline GLuint
- Purpose: Convert ARGB 1555 pixel format to RGBA 8888 with full alpha
- Inputs: InPxl (16-bit color value)
- Outputs/Return: GLuint (32-bit color with alpha = 0xFF)
- Notes: Handles both little-endian and big-endian platforms; uses FiveToEight expansion

### SetPixelOpacities[RGBA](...)
- Purpose: Update pixel alpha values based on texture options (crisp vs. blended edges)
- Calls: Invoked during texture setup to modify opacity before uploading to OpenGL

### LoadModelSkin(ImageDescriptor&, short Collection, short CLUT)
- Purpose: Load external image file as texture replacement
- Inputs: Image descriptor, collection index, color table index
- Outputs/Return: void (updates image parameter)

## Control Flow Notes

**Initialization phase**: OGL_StartTextures() initializes global texture state.

**Per-frame phase**: OGL_FrameTickTextures() is called after rendering to update usage counters and evict stale textures.

**Texture setup phase**: TextureManager::Setup() is called once per unique texture to load and cache it. Attempts LoadSubstituteTexture() first; on failure, derives geometry and allocates OpenGL buffers.

**Render phase**: RenderNormal() ΓåÆ RenderGlowing() ΓåÆ RenderBump() bind textures and configure matrix for geometry-specific scaling/offset.

**Shutdown phase**: OGL_StopTextures() releases all resources.

## External Dependencies
- **OGL_Headers.h**: OpenGL API bindings (glBindTexture, GLuint, GLenum, GLdouble)
- **OGL_Subst_Texture_Def.h**: OGL_TextureOptions, BillboardType, texture option parsing
- **scottish_textures.h**: Transfer modes, shading table definitions, shape_descriptor, polygon/rectangle structures
- **ImageDescriptorManager**: Image data wrapper with premultiplication and descriptor queries (defined elsewhere)
- **OGL_TextureOptions** (base class OGL_TextureOptionsBase): Opacity type, blending modes, bloom parameters
