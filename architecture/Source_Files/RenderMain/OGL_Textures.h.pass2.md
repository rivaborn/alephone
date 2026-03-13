# Source_Files/RenderMain/OGL_Textures.h - Enhanced Analysis

## Architectural Role

OGL_Textures.h is the GPU memory manager for the OpenGL rendering backend, implementing a stateful texture cache that bridges between Marathon's indexed-color texture references and modern GPU VRAM. It acts as a critical abstraction layer between the game world (which references textures via shape_descriptor indices) and OpenGL rendering (which requires GPU-resident texture objects). The file enables dynamic texture substitution, supports deferred texture loading, and implements per-frame garbage collection to manage VRAM under tight memory constraints typical of the Marathon engine era.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderRasterize_Shader.cpp** (`_render_node_object_helper`) ΓÇö binds textures and invokes `RenderNormal()/RenderGlowing()/RenderBump()` to set up geometry-specific texture state before rasterization
- **OGL_Render.cpp** ΓÇö uses `TextureManager::Setup()` to load textures from shape descriptors and invokes texture rendering methods during polygon/sprite rendering
- **OGL_FBO.cpp** ΓÇö may query texture dimensions and bind substitute textures for off-screen rendering
- **OGL_Subst_Texture_Def.h** consumers ΓÇö `OGL_TextureOptions` configuration is read by `TextureManager` to determine blend modes, bloom, infravision tinting
- **shapes.cpp** ΓÇö provides collection/CLUT/bitmap indices used by `TextureManager::Setup()`

### Outgoing (what this file depends on)
- **OGL_Headers.h** ΓÇö OpenGL API (glBindTexture, glTexImage2D, glGenTextures, GLenum/GLuint types)
- **OGL_Subst_Texture_Def.h** ΓÇö `OGL_TextureOptions` structure defining opacity type, blend modes, bloom parameters, billboard mode
- **scottish_textures.h** ΓÇö `TransferMode` constants (tinting, wobble, slide), `shape_descriptor` unpacking, color lookup tables
- **ImageDescriptorManager** (implicit dependency) ΓÇö holds image data, mipmap state, premultiplication flags
- **Global state**: `TxtrTypeInfoList[]` (texture format configs), `gGLTxStats` (statistics), infravision tinting state

## Design Patterns & Rationale

**Stateful TextureManager-per-texture pattern**: Unlike a global texture cache, each unique game texture gets its own `TextureManager` instance. This decouples texture lifecycle management from global cache logic and allows passing setup parameters (shape descriptor, transfer mode, etc.) as public member variables without constructor overloading. Tradeoff: multiple TextureManager instances exist temporarily during rendering, creating allocation churnΓÇömitigated by reuse in subsequent frames.

**Lazy substitution loading**: `LoadSubstituteTexture()` attempts to load external image files first (PNG, DDS); if missing, the code falls back to converting raw bitmap data. This allows artists to override low-res bitmaps without rebuilding WAD filesΓÇöa key feature for modding and modern asset pipelines.

**Pre-computed color tables**: Rather than performing indexedΓåÆRGB conversion at render time, `FindColorTables()` and `GetOGLTexture()` pre-allocate 256-color lookup tables. This trades memory for speed, befitting an engine designed for 30 FPS software rasterization where per-pixel lookups would be prohibitive.

**Per-frame usage tracking with lazy eviction**: `TextureState::unusedFrames` and `OGL_FrameTickTextures()` implement LRU-like VRAM management without explicit reference counting. Textures are evicted after N unused frames, allowing dynamic memory reclamation during level transitions or heavy texture switching. This avoids determinism-breaking allocator fragmentation.

**Multi-variant texture support**: `TextureState` allocates separate texture IDs for Normal, Glowing, and Bump variants. This avoids conditional branching in hot rendering code (each variant just calls the appropriate `Use()` method) and simplifies shader switching.

## Data Flow Through This File

```
Input (TextureManager::Setup):
  shape_descriptor ΓåÆ unpack collection/bitmap/frame indices
  ΓåÆ LoadSubstituteTexture() fails? ΓåÆ derive geometry + allocate buffers
  ΓåÆ FindColorTables() fetches shading tables from GameWorld
  ΓåÆ GetOGLTexture() converts indexed pixels to RGBA via color table
  ΓåÆ PlaceTexture() uploads to GPU via glTexImage2D

Per-frame (OGL_FrameTickTextures):
  TextureState::FrameTick() ΓåÆ increment unusedFrames counter
  ΓåÆ if unusedFrames > threshold: Reset() ΓåÆ deallocate GPU texture IDs

Render (TextureManager::RenderNormal/Glow/Bump):
  TextureState::Use() ΓåÆ mark variant in-use for this frame
  ΓåÆ glBindTexture(GL_TEXTURE_2D, TextureState::IDs[Which])
  ΓåÆ SetupTextureMatrix() applies U_Scale/V_Scale/U_Offset/V_Offset
  ΓåÆ geometry rendering samples from bound texture
```

**Key state transitions**: Unallocated ΓåÆ Allocate() ΓåÆ (Normal|Glow|Bump generated lazily) ΓåÆ per-frame Use() resets age ΓåÆ evicted if unused. Substitute textures bypass geometry derivation entirely, reducing CPU cost for modded content.

## Learning Notes

- **Indexed-color heritage**: The engine predates true-color graphics APIs. Textures are stored as palette indices + shading tables (brightness variations). `Convert_16to32()` and `FindColorTables()` convert this to RGBA for OpenGL, showing how legacy art assets drove architecture.
- **Endianness-aware pixel packing**: `Convert_16to32()` branches on `PlatformIsLittleEndian()`, a pattern now obsolete but essential for Mac/PC compatibility in the 2000s.
- **Landscape special case**: `GetFakeLandscape()` and `Landscape_AspRatExp` handle dynamic terrain textures with custom aspect ratio scalingΓÇöa runtime optimization for outdoor levels.
- **Silhouette/infravision modes**: `ModifyCLUT()`, `FindInfravisionVersionRGBA()`, and `FindSilhouetteVersion()` implement special rendering modes (thermal vision, outlines) via color table substitutionΓÇöpredates modern post-process effects.
- **Texture coordinate offset/scale for padding**: Sprites are often padded to power-of-two dimensions for GPU efficiency; `U_Scale`, `V_Offset` hide the padding from the rasterizerΓÇöelegant math.

## Potential Issues

1. **Unvalidated public member state**: `TextureManager` takes public members (`ShapeDesc`, `Texture`, `ShadingTables`) that must be initialized before `Setup()`. There's no guard against calling `RenderNormal()` before `Setup()` completes, risking use-of-uninitialized state. Consider making Setup() a constructor parameter or adding an `is_setup` flag.

2. **Eviction policy sensitivity**: Texture eviction after N unused frames can cause performance cliffs if N is too low (thrashing on level-transition filtering) or wastes VRAM if too high. Frame-count-based eviction also couples to frame rate; a 60-FPS port may evict faster than intended.

3. **Memory fragmentation with many collections**: If a map loads many collection bitmaps, `CollBitmapTextureState` allocates `TextureState[NUMBER_OF_OPENGL_BITMAP_SETS]` per collection, leading to fragmented heap allocations. A single vector with hash-keyed lookup would reduce fragmentation.

4. **Color table coupling**: `NormalColorTable[]` and `GlowColorTable[]` are allocated and pre-computed for every texture, even if not used. Glow-mapped textures are relatively rare; lazy color table computation would save memory for non-glowing textures.
