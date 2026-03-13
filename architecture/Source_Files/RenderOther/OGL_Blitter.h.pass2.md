# Source_Files/RenderOther/OGL_Blitter.h - Enhanced Analysis

## Architectural Role

`OGL_Blitter` is a specialized 2D OpenGL image renderer that sits in the *RenderOther* subsystemΓÇödistinct from the main 3D rendering pipeline in *RenderMain*. It provides an OpenGL backend for the `Image_Blitter` interface, enabling tiled texture-based rendering of 2D imagery (UI, HUD, menus, overlays) with a critical responsibility: recovering GL resources after context loss via static registry coordination. The class bridges the abstract blitter interface (supporting software/GL/shader backends) and low-level OpenGL texture lifecycle.

## Key Cross-References

### Incoming (who depends on this file)
- **Screen drawing system** (`RenderOther/screen_drawing.cpp`): The `_set_port_to_*` family and draw functions likely instantiate or use blitters for compositing HUD/UI
- **ImageLoader subsystem** (`RenderMain/ImageLoader*`): Provides decoded image data to populate textures
- **OGL_Render / OGL_Textures** (`RenderMain/OGL_*.cpp`): Related OpenGL infrastructure; may coordinate context-loss callbacks with the registry
- **render.h/cpp** (`RenderMain/render.cpp`): Frame rendering orchestrator; calls `BoundScreen()` for viewport setup

### Outgoing (what this file depends on)
- **Image_Blitter** (base class in `RenderOther/Image_Blitter.h`): Inherits `crop_rect`, tint, rotation, and surface-management abstractions
- **ImageLoader.h**: Implied for loading source images (though not directly visible in header)
- **OGL_Headers.h**: OpenGL type definitions (`GLuint`) and platform-specific GL initialization
- **screen.cpp functions**: Likely calls `BoundScreen()`, `WindowToScreen()`, `ScreenWidth()`, `ScreenHeight()` implemented nearby for viewport management and coordinate transforms

## Design Patterns & Rationale

| Pattern | Evidence | Rationale |
|---------|----------|-----------|
| **Static Registry** | `m_blitter_registry` tracks all active instances | Enables safe global texture reload on GL context lossΓÇöcritical for windowed rendering and Alt+Tab recovery |
| **Lazy Loading** | `_LoadTextures()` called implicitly on first `Draw()` | Defers expensive GPU upload until actually needed; avoids double-allocation |
| **Tiling Strategy** | Fixed `tile_size = 256` divides large images | Reflects era-specific GPU texture size limits; 256├ù256 was a safe maximum for older GL implementations |
| **Polymorphic Rendering** | Inherits `Image_Blitter`; overrides `Draw()` | Multiple backends (software rasterizer, GL, shaders) swap in via virtual dispatch; header-only polymorphism |
| **Configurable Filtering** | `nearFilter` parameter in constructor | Different rendering contexts need different filter modes (e.g., UI needs sharp GL_NEAREST, some effects need GL_LINEAR) |

**Rationale for structure**: This design accommodates *historical GPU constraints* (texture size limits, context loss on window focus) and *backend flexibility* (software fallback if GL unavailable). The registry is essential because OpenGL context loss is asynchronousΓÇö`StopTextures()` signals global recovery; the registry walks all instances to reload.

## Data Flow Through This File

1. **Initialization**: Constructor registers instance in global `m_blitter_registry`; textures remain unloaded.
2. **First Draw**: `Draw(Image_Rect dst, Image_Rect src)` checks `m_textures_loaded` ΓåÆ calls `_LoadTextures()` if false.
3. **_LoadTextures**: 
   - Queries source image dimensions (from inherited `Image_Blitter` state or external source)
   - Divides image into 256├ù256 tiles ΓåÆ calculates `m_rects` (SDL_Rect for each tile)
   - Uploads each tile to GPU ΓåÆ stores GLuint handle in `m_refs`
4. **Rendering**: `Draw()` iterates `m_refs`, issues OpenGL texture-bind and render calls for visible tiles.
5. **Context Loss**: `StopTextures()` (global static call) ΓåÆ iterates `m_blitter_registry` ΓåÆ calls `_UnloadTextures()` on each instance.
6. **Reload**: Next `Draw()` call detects `m_textures_loaded == false` ΓåÆ `_LoadTextures()` re-uploads.
7. **Shutdown**: Destructor calls `Unload()` ΓåÆ `_UnloadTextures()` ΓåÆ `Deregister(this)` removes from registry.

## Learning Notes

**What a developer learns from this file:**
- **Era-specific constraints**: 256├ù256 tiling is not arbitraryΓÇöreflects GPU memory/addressing limits of the ~2006 era (when written). Modern engines use dynamic texture atlasing and streaming.
- **Context loss handling**: Static registry pattern is a pragmatic solution for synchronous global resource recoveryΓÇömodern engines use resource managers with reference counting and per-frame defrag passes.
- **Polymorphic backends**: The `Image_Blitter` hierarchy shows intentional architecture to support multiple rendering paths (software, GL, shader-based) without recompiling.
- **Lazy initialization**: Deferring `_LoadTextures()` to first use reduces startup time and memory pressure.

**Idiomatic to this engine / era that differs from modern engines:**
- Manual texture lifecycle management (no RAII resource wrappers; `Unload()` must be called explicitly)
- Global static state for context coordination (modern engines use message buses or centralized resource managers)
- SDL_Surface as an interchange format (modern engines use opaque GPU textures or intermediate image views)
- No apparent thread safety on registry (assumes single-threaded rendering; modern engines protect concurrent resource access with mutexes)

## Potential Issues

1. **Static registry memory safety**: If a blitter instance is deleted while `m_blitter_registry` points to it, a subsequent `StopTextures()` call will dereference a dangling pointer. Relies on correct destructor ordering and no double-deletion.
2. **No thread safety**: `m_blitter_registry` is accessed without synchronization. If rendering and resource recovery happen on different threads, race conditions are possible.
3. **Implicit context assumptions**: `BoundScreen()` and OpenGL rendering calls assume a valid GL context is current; no validation that context matches the textures' GPU owner.
4. **Tiling mismatch**: If image dimensions change after first load but before `_UnloadTextures()`, tile boundaries in `m_rects` may be stale.
