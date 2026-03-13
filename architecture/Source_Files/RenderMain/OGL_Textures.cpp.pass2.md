# Source_Files/RenderMain/OGL_Textures.cpp - Enhanced Analysis

## Architectural Role

OGL_Textures is the **GPU memory manager** for the rendering pipeline, responsible for translating high-level shape descriptors into OpenGL texture objects and managing their lifecycle within constrained VRAM. It sits at the critical junction between the visibility/object placement stages (RenderVisTree, RenderPlaceObjs) and the final rasterization backends (OGL_Render, RenderRasterize_Shader). The frame-tick purging mechanism directly addresses 2000s-era GPU memory constraints (32ΓÇô256 MB); without it, long gameplay sessions would exhaust VRAM and force expensive full cache flushes.

## Key Cross-References

### Incoming (who depends on this file)
- **OGL_Render.cpp** ΓåÆ calls `TextureManager::Setup()` and `RenderNormal/Glowing/Bump()` per polygon/sprite each frame; manages texture matrix transforms
- **OGL_Blitter.cpp** ΓåÆ loads UI textures via `LoadModelSkin()`; font textures via `FontSpecifier::OGL_ResetFonts()`
- **RenderRasterize_Shader.cpp** ΓåÆ binds textures for shader-based rendering path (modern GPU path)
- **OGL_Setup.cpp** ΓåÆ calls `OGL_StartTextures()` at initialization; coordinates GL context creation
- **Preferences system** ΓåÆ provides `OGL_ConfigureData` (filter/format choices) via `Get_OGL_ConfigureData()`

### Outgoing (what this file depends on)
- **Image loading** ΓåÆ `ImageDescriptor`, `ImageDescriptorManager` via `LoadSubstituteTexture()`; handles DDS/PNG/BMP decompression
- **Collection system** (shapes.cpp) ΓåÆ `get_bitmap_index()`, `get_collection_colors()`, shape descriptor parsing
- **OpenGL API** ΓåÆ `glGenTextures`, `glBindTexture`, `glTexImage2D`, `gluBuild2DMipmaps`, `glCompressedTexImage2DARB`, `glTexParameteri`
- **Global state** ΓåÆ reads `InfravisionActive`, `IVDataList[]` (effects state); writes `gGLTxStats` (metrics)

## Design Patterns & Rationale

**Object Pool + Active List**: TextureStateSets[][] pre-allocates all possible texture states at startup (one per bitmap per collection per type). The separate `sgActiveTextureStates` linked list tracks *only* allocated textures, enabling efficient O(n_active) frame-tick updates rather than O(total_capacity).

**Frame-Tick Purging**: Aggressive VRAM recycling (walls 300 frames Γëê10s, sprites 450 frames Γëê15s) reflects constrained GPU memory. Landscapes never purge because they're persistent world geometry. This differs sharply from modern dynamic VRAM managers.

**Substitution Pattern**: `LoadSubstituteTexture()` bypasses bitmapΓåÆpower-of-2 padding logic, enabling mod-friendly runtime texture replacement without recompilation.

**Deferred Effect Application**: Infravision/silhouette effects are baked into texture data during `Setup()`, not per-frame blending. This is efficient (immutable GPU data) but inflexible (can't toggle effects mid-render).

**Per-Type Configuration Isolation**: Separate `TxtrTypeInfoList[type]` allows walls and UI to use different filters/formats (e.g., anisotropic for walls, nearest-neighbor for UI fonts), solving cross-cutting concerns without runtime branching.

## Data Flow Through This File

1. **Input**: `TextureManager::Setup()` receives shape descriptor + transfer mode ΓåÆ parses collection/clut/frame indices
2. **Lookup**: Queries `TextureStateSets[TextureType][Collection][Bitmap]` for cached state
3. **Allocation**: If new, calls `TextureState::Allocate()` ΓåÆ `glGenTextures()` + linked-list registration
4. **Format Conversion**: `FindColorTables()` converts Marathon shading tables (ARGB 5551/8888) ΓåÆ OpenGL RGBA 8888 with opacity adjustments
5. **Upload**: `PlaceTexture()` calls `glTexImage2D()` or `glCompressedTexImage2DARB()` + filtering/wrapping setup
6. **Per-Frame**: `OGL_FrameTickTextures()` iterates active list, increments `unusedFrames`, calls `Reset()` when threshold exceeded
7. **Output**: GPU texture IDs bound via `glBindTexture()`, statistics accumulated in `gGLTxStats`

## Learning Notes

- **Era-Appropriate Design**: Manual VRAM management (2000ΓÇô2010s era) before unified GPU memory architecture. Modern engines use budgeted pools or virtual texturing.
- **No Atlasing**: Each bitmap ΓåÆ separate GPU texture. Fragmentation risk; modern equivalent would batch into GPU atlases to reduce state changes.
- **Substitution as Modding**: `LoadSubstituteTexture()` shows pre-asset-pipeline modding: runtime file replacement for textures, no recompilation needed. Precursor to modern mod APIs.
- **Unusual State Partitioning**: Three texture IDs per bitmap (Normal, Glowing, Bump) is atypical; most modern engines use multi-channel textures (RGB + alpha + normal maps).
- **Infravision as CPU Effect**: Tinting applied during texture upload, not GPU shader. Reflects era before ubiquitous GLSL (2004ΓÇô2008 transition).

## Potential Issues

- **No VRAM Budget Enforcement**: `gGLTxStats` tracks metrics but never blocks allocation if over budget; pure LRU purging with no priority weighting.
- **Purge-Induced Stutter**: Aggressive thresholds (10ΓÇô20s) may cause frame hitches if high-frequency textures are evicted and reloaded; no adaptive tuning.
- **Format Fragmentation**: Supporting RGBA8/RGBA4/RGBA2 without atlasing causes GPU cache misses; modern GPUs prefer uniform formats + compression (ASTC, BC7).
- **Silhouette/Infravision Exclusivity**: Effects are mutually exclusive at setup time; cannot blend or layer them, limiting visual flexibility.
