# Source_Files/RenderMain/OGL_FBO.cpp - Enhanced Analysis

## Architectural Role

This file is the foundation for GPU-accelerated post-processing effects in the Aleph One rendering pipeline. FBOs enable off-screen renderingΓÇöthe backbone of the shader-based rasterizer (`Rasterizer_Shader`) where effects like blur, bloom, and multisampling antialias require ping-pong buffering between intermediate textures. The file bridges the fixed-function OpenGL layer (`OGL_Setup`, `OGL_Render`) and higher-level effect orchestration, enabling iterative multi-pass rendering that would be impossible with direct-to-screen drawing.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderRasterize_Shader.cpp** ΓåÆ Uses `FBO` and `FBOSwapper` for post-processing passes (bloom, blur effects); allocates FBOs per-effect
- **Rasterizer_Shader.h/cpp** ΓåÆ Manages FBO lifecycle during shader-based rasterization; calls `activate()`/`deactivate()` to switch render targets
- **OGL_Faders.h/cpp** ΓåÆ May leverage FBOs for fade effects (tint, static, dodge, burn) rendering to intermediate targets
- **Global sRGB state** (`Using_sRGB` from `OGL_Setup.h`) ΓåÆ Read by `deactivate()` to restore color-space state after nested FBO pops

### Outgoing (what this file depends on)
- **OGL_RenderTexturedRect()** (`OGL_Render.h`) ΓåÆ Called by `FBO::draw()` to render FBO texture contents to screen
- **OGL_Setup.h** ΓåÆ Reads `Using_sRGB` global flag; accesses `TxtrTypeInfoList[OGL_Txtr_HUD]` for filter settings (MAG/MIN_FILTER)
- **OpenGL EXT extension functions** ΓåÆ `glGenFramebuffersEXT`, `glBindFramebufferEXT`, `glCheckFramebufferStatusEXT`, `glFramebufferTexture2DEXT` (legacy ARB/EXT API; assumes EXT framebuffer_object extension)

## Design Patterns & Rationale

**Stack-Based Activation Pattern**: `active_chain` maintains a LIFO stack of active FBOs, enabling nested rendering contexts without manual state restoration. Example: render to FBO_A, then activate FBO_B (pushed), perform effect, deactivate FBO_B (pops), continue with FBO_A. This mirrors immediate-mode GL's viewport/matrix stack discipline, avoiding manual rebind boilerplate.

**Why not true RAII?** Activation is explicit (`activate()`/`deactivate()` calls), not constructor/destructor-based. This decouples FBO lifetime (allocation) from activation lifetime (usage), allowing one FBO to be used multiple times or shared across frames. The downside: destructor doesn't pop from `active_chain` if FBO is still activeΓÇöa potential dangling-pointer bug if the FBO is deleted while on the stack.

**Ping-Pong Double-Buffering via FBOSwapper**: Rather than reading-and-writing the same FBO (which creates read-after-write hazards in GPU pipelines), `FBOSwapper` alternates between two FBOs. Each effect pass reads from one, writes to the other, then swaps. This avoids GPU stalls and is essential for iterative effects (e.g., multi-tap blur, cascading filters).

**sRGB Plumbing**: Color-space state is threaded through activation/deactivation (`_srgb` member, `Using_sRGB` global). Each FBO can enforce its own sRGB mode, and `deactivate()` restores the *previous* FBO's setting or the global default. This supports hybrid pipelines mixing sRGB and linear-space rendering.

## Data Flow Through This File

**Initialization Phase**:
1. `FBO` constructor allocates GPU objects: framebuffer, depth renderbuffer, color texture (GL_TEXTURE_RECTANGLE_ARB)
2. Texture filtering copied from HUD texture settings (`TxtrTypeInfoList[OGL_Txtr_HUD].FarFilter/NearFilter`)
3. Framebuffer completeness asserted; crashes if GPU rejects the configuration

**Rendering Phase** (typical post-processing loop):
1. Game world rendered to FBO_A (via `activate()`, draw geometry, deactivate)
2. `FBOSwapper::filter()` ΓåÆ activate FBO_B, call `draw()` to sample FBO_A's texture, apply shader, write to FBO_B, `swap()`
3. Loop: `filter()` repeatedly applies cascading effects (e.g., Gaussian blur in multiple passes)
4. Final result in whichever FBO is "current" (toggled by `draw_to_first` flag)

**Output Phase**:
1. `FBOSwapper::draw()` ΓåÆ Activates inactive FBO, calls `draw_full()` (prepare 2D ortho mode, render texture to screen, restore matrices)
2. sRGB is disabled if rendering to screen framebuffer (linear output assumed)

**Special Case: Multisampling Antialias**:
- `blend_multisample()` binds source FBO texture to unit 1, configures hardcoded texture coordinates from source dimensions, performs multi-tap blend with unit 0
- Designed for antialiasing via hardware-generated sub-pixel samples or gather-based filtering

## Learning Notes

**Era & Legacy**: This is **OpenGL 1.4ΓÇô2.0 era code** (EXT framebuffer_object, fixed-function pipeline assumptions). Modern engines use core GL 3.3+ with vertex/fragment shaders, compute shaders, and compute-based post-processing. The matrix-stack-based projection setup (`glMatrixMode`, `glPushMatrix`, `glLoadIdentity`, `glOrtho`) is emblematic of fixed-function OpenGLΓÇönow considered an anti-pattern (view matrices should be uploaded as uniform buffers).

**GL_TEXTURE_RECTANGLE_ARB**: Non-power-of-two rectangle textures, predating the NPOT extension. This avoids the "expand to next POT, then shrink in shader" workaround, saving VRAM.

**Assertion on Completeness**: `glCheckFramebufferStatusEXT()` asserts in constructor but silently skips redundant activation in `activate()`. This is defensive against GPU driver bugs at startup but doesn't validate state during rendering.

**State Management Asymmetry**: `activate()` *sets* sRGB state based on `_srgb` member. `deactivate()` *restores* it from the previous FBO or global. This works because the stack discipline ensures state is always paired, but it's fragile if nesting depth exceeds 2ΓÇô3 levels or if state is modified externally.

## Potential Issues

1. **Destructor Dangling Pointer**: If an FBO is deleted while still on `active_chain`, subsequent `deactivate()` calls will dereference a freed pointer. No guard check in destructor.

2. **texID Texture Leak**: Constructor allocates `texID` via `glGenTextures()`, but destructor only deletes framebuffer and renderbuffer, not the texture. The texture may leak if not cleaned up by OGL_Textures cache management.

3. **blend_multisample Inflexibility**: Hardcoded texture coordinates `{0, _h, _w, _h, _w, 0, 0, 0}` assume source FBO dimensions match at runtime. Dynamic resizing (window resize, resolution change) without re-creating the FBO will cause UV misalignment.

4. **Prepare/Reset Asymmetry**: `prepare_drawing_mode()` pushes matrices and conditionally disables blend; `reset_drawing_mode()` *always* enables blend and depth test, ignoring the caller's intent. If the caller wanted blending disabled for a reason, it's unconditionally re-enabledΓÇöa potential visual bug.
