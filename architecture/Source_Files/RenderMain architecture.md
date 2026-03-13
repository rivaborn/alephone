# Subsystem Overview

## Purpose

RenderMain orchestrates the rendering pipeline for Aleph One, transforming 3D game world geometry and objects into 2D screen output. It coordinates visibility determination, polygon sorting, object placement, and rasterization across multiple backends (software rasterizer, OpenGL classic, and modern shader-based GPU rendering).

## Key Files

| File | Role |
|------|------|
| `render.cpp` / `render.h` | Central rendering orchestration: view setup, effect management, pipeline coordination, frame rendering entry point |
| `RenderVisTree.cpp` / `.h` | Visibility determination via ray casting to build hierarchical tree of visible polygons |
| `RenderSortPoly.cpp` / `.h` | Depth-based polygon sorting with clipping window calculation |
| `RenderPlaceObjs.cpp` / `.h` | Object placement, projection, visibility culling, and depth-sorting into render tree |
| `RenderRasterize.cpp` / `.h` | Polygon clipping and coordinate transformation for rasterization |
| `RenderRasterize_Shader.cpp` / `.h` | Shader-based GPU rasterization with multi-pass rendering and post-processing |
| `Rasterizer.h` | Abstract rasterizer interface defining rendering contract |
| `Rasterizer_SW.h` | Software rasterizer implementation (Scottish Textures) |
| `Rasterizer_OGL.h` | OpenGL classic rasterizer adapter |
| `Rasterizer_Shader.h` / `.cpp` | Shader-based OpenGL rasterizer with FBO management |
| `scottish_textures.cpp` / `.h` | Software texture rasterizer: perspective-correct texture mapping, DDA line tables, transfer modes |
| `AnimatedTextures.cpp` / `.h` | Animated wall textures: frame sequencing, tick-based updates, XML configuration |
| `OGL_Render.cpp` / `.h` | OpenGL rendering backend: context lifecycle, coordinate transforms, 3D sprite/model rendering |
| `OGL_Setup.cpp` / `.h` | OpenGL initialization, extension detection, configuration, resource loading |
| `OGL_Shader.cpp` / `.h` | GLSL shader compilation, linking, program management, uniform binding |
| `OGL_FBO.cpp` / `.h` | OpenGL framebuffer object management for off-screen rendering and ping-pong buffering |
| `OGL_Textures.cpp` / `.h` | OpenGL texture lifecycle: loading, caching, VRAM management, effects (infravision, silhouettes) |
| `OGL_Model_Def.cpp` / `.h` | 3D model loading, storage, texture associations, MML configuration |
| `OGL_Subst_Texture_Def.cpp` / `.h` | Substitute texture configuration for walls and sprites |
| `OGL_Faders.cpp` / `.h` | Fade effect rendering (tint, static, negation, dodge, burn) |
| `ImageLoader.h` / `ImageLoader_SDL.cpp` / `ImageLoader_Shared.cpp` | Image file loading (DDS, PNG, BMP), format conversion, mipmap management |
| `Crosshairs.h` / `Crosshairs_SDL.cpp` | Crosshair HUD rendering |
| `shapes.cpp` | Shape collection loading, bitmap decompression, shading table construction |
| `shape_definitions.h` / `shape_descriptors.h` | Shape collection metadata and 16-bit packed descriptor encoding |
| `textures.cpp` / `textures.h` | Bitmap metadata, row address tables, pixel remapping |
| `low_level_textures.h` | Template-based texture rasterization for polygons |
| `SW_Texture_Extras.cpp` / `.h` | Software texture opacity table management |
| `collection_definition.h` | Binary-compatible structures for Marathon shape collections |
| `DDS.h` | DirectDraw Surface format structures |
| `OGL_Headers.h` | Cross-platform OpenGL header abstraction |
| `vec3.h` | Vector and matrix types for 3D graphics |

## Core Responsibilities

- **View and camera management**: Initialize and update camera position, orientation, FOV, and screen-to-world projection each frame
- **Visibility determination**: Build hierarchical visibility tree via ray casting through 2D map geometry from player viewpoint
- **Polygon sorting**: Depth-order visible polygons into rendering tree with clipping window calculation
- **Object placement**: Convert game objects into renderer-ready structures with projection, culling, and depth sorting
- **Rasterization**: Clip polygons to viewport, transform to screen space, dispatch to backend rasterizer
- **Multi-backend support**: Abstract rasterizer interface supporting software, OpenGL, and shader-based rendering
- **Texture management**: Handle texture loading, caching, VRAM lifecycle, and animation
- **Visual effects**: Apply transfer modes (slides, wobbles, tints, fades), explosions, teleports, camera shake
- **3D model rendering**: Support loaded 3D models with lighting, skins, and transformations
- **HUD rendering**: Render weapons, crosshairs, and UI elements in foreground layer
- **Performance optimization**: Track texture usage, purge unused VRAM, preload textures

## Key Interfaces & Data Flow

**Exposes to Game Engine:**
- `render.h` function prototypes: view initialization, frame rendering, effect triggering
- `view_data` structure: camera position, orientation, FOV, screen dimensions, effect state
- Backend selection via `graphics_preferences` (software / OpenGL / shader modes)

**Consumes from Game Engine:**
- Game world from `map.h`: polygons, lines, endpoints, sides, platforms, adjacency
- Player state from `player.h`: camera position, orientation, weapon info
- Object state from `world.h`: inhabitants/objects, ephemera, light sources, media
- Configuration from `preferences.h`: rendering backend, texture quality, alpha blending mode, gamma
- Visual effects from `ViewControl.h`: FOV modes, view distortion, effect phases

**Cross-subsystem Dependencies:**
- **Files subsystem**: Shape/texture loading and caching
- **Audio subsystem**: Synchronized visual effects
- **Input subsystem**: View control during rendering frame
- **UI/Screen subsystem**: Framebuffer binding, display surface access, font rendering

## Runtime Role

**Initialization (`allocate_render_memory`, `initialize_render_classes`):**
- Allocate visibility tree, polygon sorting, object placement data structures
- Initialize rasterizer backend based on graphics preferences
- Load and compile OpenGL shaders (if enabled)
- Initialize framebuffer objects and texture managers

**Frame Rendering (`render_view`, invoked each game tick):**
1. Update view state: yaw/pitch vectors, FOV transitions, vertical scaling
2. Apply render effects: explosion shake, teleport fold, transfer mode animations
3. Build visibility tree: ray cast from viewpoint to determine visible polygons
4. Sort visible polygons: depth-order with clipping window calculation
5. Place game objects: project objects into render tree with visibility culling
6. Rasterize scene: traverse sorted tree, clip polygons, render surfaces and objects
7. Render foreground: weapon sprites and HUD elements
8. Composite effects: render crosshairs, faders, visual feedback

**Shutdown:**
- Release rasterizer resources
- Unload OpenGL shaders and framebuffer objects (if used)
- Free texture and model caches

## Notable Implementation Details

- **Multi-pass rendering**: Shader-based renderer supports diffuse and glow passes for bloom effects
- **Visibility system**: Original Marathon ray-casting algorithm adapted for BSP tree traversal with screen-edge clipping
- **Rasterization templates**: Software renderer uses compile-time template dispatch on pixel bit depth (8/16/32) for performance
- **DDA line tables**: Digital Differential Analyzer precalculation for perspective-correct texture mapping
- **Framebuffer object ping-ponging**: Double-buffered FBO swapping for post-processing pipelines (filtering, blur, blending)
- **Animated textures**: Frame sequencing with bidirectional support, updated each tick before rasterization
- **sRGB color space**: Conditional linear-to-sRGB conversion for physically correct rendering
- **Substitute textures**: Per-collection, per-CLUT texture replacement with cascading fallback lookup (exact ΓåÆ infravision ΓåÆ silhouette ΓåÆ default)
- **Transfer mode support**: Texture slides, wobbles, pulsates, tints, static, and randomization effects
- **Model skinning**: Multiple texture variants (normal/infravision/silhouette) per model with per-CLUT customization
- **Software path optimization**: Lazy texture loading, frame-tick-based VRAM purging, texture state caching
- **Marathon 1 compatibility**: Optional exploration mission support via background polygon visibility checking
