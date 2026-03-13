# Source_Files/RenderOther/OGL_Blitter.cpp - Enhanced Analysis

## Architectural Role

OGL_Blitter is the **2D overlay renderer** for the engine's UI, HUD, and menu systems, bridging SDL software rendering and OpenGL. It sits in RenderOther (alongside `screen_drawing.cpp`, `computer_interface.cpp`, `overhead_map.cpp`) forming a parallel compositing pipeline to the 3D world renderer in RenderMain. While RenderMain handles 3D polygon/model rasterization, OGL_Blitter handles arbitrary 2D image tiling and GPU upload for non-world UI elements (crosshairs, terminals, motion sensors, pause screens, etc.).

## Key Cross-References

### Incoming (who depends on this file)
- **Screen UI layer** (`screen_drawing.cpp`, `computer_interface.cpp`, `overhead_map.cpp`) ΓÇö likely instantiate OGL_Blitter subclasses or delegate to Draw() for UI compositing
- **Image_Blitter parent class** ΓÇö abstract interface defining Loaded(), ScreenWidth/Height(), BoundScreen(), WindowToScreen()
- **Rendering orchestration** ΓÇö called after 3D world render but before frame swap (typical UI overlay position)

### Outgoing (what this file depends on)
- **OGL_Render.h** ΓÇö `OGL_RenderTexturedRect()` helper for immediate-mode geometry submission
- **OGL_Setup.h** ΓÇö `Get_OGL_ConfigureData()` config flags, `PlatformIsLittleEndian()` byte-order detection
- **Screen singleton** (`screen.h`) ΓÇö screen bounds, window-to-screen coordinate mapping (supports windowed/fullscreen modes)
- **OpenGL global state** ΓÇö GL_TEXTURE_2D, texture objects, blending, matrix stack
- **SDL2** ΓÇö surface creation, pixel blitting, RGBA format marshaling

## Design Patterns & Rationale

**Registry + Bulk Cleanup**: Static `m_blitter_registry` and `StopTextures()` implement a factory cleanup hook. When GL context is lost (mode switch, minimization), `StopTextures()` iterates all active blitters and calls `_UnloadTextures()` in bulk. This is era-appropriate (pre-OpenGL 3.3+ robustness extensions); modern engines use persistent mapped buffers or handle context loss more gracefully.

**Lazy Texture Loading**: Textures created on first `Draw()` call, not in constructor. Allows deferred resource allocation until actually needed (small memory footprint for unused UI elements). Early guard `if (!Loaded()) return` prevents loading if parent hasn't set m_surface.

**Tiling/Chunking for GPU Constraints**: Power-of-two tiles (max 256├ù256) decompose arbitrary SDL surfaces into GPU-friendly chunks. This reflects early 2000s hardware constraints (non-POT textures less efficient or unsupported). The `TextureFix` config option floors tile size to 128 ΓÇö likely a workaround for buggy vendor drivers or mobile GPUs.

**Edge Pixel Smearing**: Replicated border pixels prevent visible seams when GPU samples beyond actual content due to linear filtering. This is a classic GPU texturing artifact mitigation (alternative: use GL_CLAMP_TO_EDGE + careful UV math).

## Data Flow Through This File

```
SDL_Surface (m_surface)
  Γåô
[_LoadTextures: tiling]
  Γö£ΓöÇ decompose into power-of-two tiles
  Γö£ΓöÇ replicate edge pixels horizontally/vertically
  ΓööΓöÇ upload per-tile to GPU via glTexImage2D
  Γåô
m_refs (array of GL texture IDs) + m_rects (tile bounds)
  Γåô
[Draw: per-frame rendering]
  Γö£ΓöÇ transform raw_src (scaled-space) ΓåÆ src (source-space) if scaling
  Γö£ΓöÇ compute x/y scale factors (dst vs src dimensions)
  Γö£ΓöÇ frustum-cull tiles outside source region
  Γö£ΓöÇ compute normalized texture coords (V/U) per tile
  ΓööΓöÇ submit quads via OGL_RenderTexturedRect
  Γåô
Screen framebuffer (blended with GL_SRC_ALPHA / GL_ONE_MINUS_SRC_ALPHA)
```

Key state transitions: `Loaded() ΓåÆ _LoadTextures() ΓåÆ m_textures_loaded=true ΓåÆ Draw() ΓåÆ _UnloadTextures() ΓåÆ m_textures_loaded=false`

## Learning Notes

- **SDL ΓåÆ GPU bridge idiom**: This shows how early Aleph One versions handled 2D rendering pre-SDL_gpu ΓÇö manually marshal SDL_Surface pixels into GL textures for GPU upload.
- **Screen space vs. texture space**: The distinction between `raw_src` (logical, in scaled coordinate space) and `src` (physical, in source-space pixels) reflects separate display scaling and rendering scales ΓÇö important for responsiveness in windowed vs. fullscreen.
- **Immediate-mode GL**: Uses glPushAttrib/glPopAttrib, glMatrixMode, glTranslatef ΓÇö hallmarks of fixed-function OpenGL (pre-3.3+). Modern engines would use VAOs, UBOs, and shaders.
- **Rotation via matrix stack**: Rotation center at dst center is computed correctly with translate-rotate-translate pattern, avoiding gimbal lock and coordinate system confusion.

## Potential Issues

1. **Hard-coded 256-pixel tile limit** ΓÇö for very large images (e.g., 2048├ù2048 monitor-resolution overlays), this generates 64+ tiles, increasing draw calls and register pressure. Modern engines would use larger tiles or tiling only when necessary.

2. **No thread-safety on m_blitter_registry** ΓÇö Registry insertion/erasure during concurrent StopTextures() iteration could corrupt the set. However, this likely assumes single-threaded GL use (Aleph One's rendering thread owns GL context).

3. **Edge smearing masks precision** ΓÇö Replicated pixels can blur text or fine detail at tile boundaries if source has low effective DPI relative to tile size.

4. **Scaled-space coordinate transform overhead** ΓÇö Every Draw() call recomputes src coordinates from raw_src if `m_src.w != m_scaled_src.w`. Could cache if scaling is static (it likely is for HUD elements).
