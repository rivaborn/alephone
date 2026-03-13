# Source_Files/RenderMain/OGL_Setup.cpp

## File Purpose
Implements OpenGL initialization, configuration management, and resource loading for the Aleph One game engine. Handles extension detection, texture/model loading, progress tracking during resource loading, and XML/MML-based configuration parsing for fog and rendering parameters.

## Core Responsibilities
- Initialize and detect OpenGL presence on the host system
- Validate OpenGL extension support (platform-specific: GLEW on Windows, direct glGetString on Unix)
- Manage default rendering configuration (texture filtering, resolution, flags, colors)
- Load and unload texture/model resources per game collection
- Track loading progress with visual feedback (load screen or progress dialog)
- Parse and apply OpenGL settings from MML/XML configuration files
- Manage fog parameters (color, depth, mode) with backup/restore capability
- Provide sRGB color value conversion wrappers for color functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_ConfigureData` | struct | Holds all OpenGL rendering configuration (textures, models, flags, colors, vsync, sRGB settings) |
| `OGL_Texture_Configure` | struct | Per-texture-type configuration (filter, resolution, color format, max size) |
| `OGL_FogData` | struct | Fog parameters (color, depth, start, mode, landscape blend) |
| `OGL_TextureOptionsBase` | class | Base class for loading/unloading texture image data from disk |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_OGL_IsPresent` | bool | static | Cached flag indicating OpenGL availability |
| `Using_sRGB`, `Wanting_sRGB`, `Bloom_sRGB`, `FBO_Allowed`, `npotTextures` | bool | global extern | Feature availability flags (sRGB, framebuffer objects, non-power-of-two textures) |
| `FogData[OGL_NUMBER_OF_FOG_TYPES]` | OGL_FogData[] | static | Current fog configurations for above/below-liquid environments |
| `OriginalFogData` | OGL_FogData* | static | Backup of original fog data for MML reset |
| `DefaultLscpColors[4][2]` | const RGBColor[][] | static | Default landscape colors (day/night/moon/space ├ù ground/sky) |
| `ogl_progress`, `total_ogl_progress` | int | static | Current and total loading progress |
| `show_ogl_progress` | bool | static | Whether progress indication is active |
| `glMaxTextureSize` | GLint | static | Maximum texture dimension supported by GPU |
| `hasS3TC` | bool | static | Whether S3TC/DXT compression extensions are available |

## Key Functions / Methods

### OGL_Initialize
- Signature: `bool OGL_Initialize()`
- Purpose: Initialize OpenGL and detect its presence on the system
- Inputs: None
- Outputs/Return: `true` if OpenGL is present and usable; `false` otherwise
- Side effects: Sets static flag `_OGL_IsPresent`
- Calls: (conditional) `glewInit()` on Windows
- Notes: GLEW initialization commented out on Windows; behavior may differ based compile-time conditionals

### OGL_IsPresent
- Signature: `bool OGL_IsPresent()`
- Purpose: Query whether OpenGL initialization succeeded
- Inputs: None
- Outputs/Return: Cached `_OGL_IsPresent` flag
- Side effects: None
- Calls: None

### OGL_CheckExtension
- Signature: `bool OGL_CheckExtension(const std::string extension)`
- Purpose: Detect whether a named OpenGL extension is supported
- Inputs: Extension name string (e.g., "GL_ARB_texture_compression")
- Outputs/Return: `true` if extension is available
- Side effects: None (read-only queries)
- Calls: `glewIsSupported()` on Windows; `glGetString(GL_EXTENSIONS)` on Unix platforms
- Notes: Cross-platform wrapper; manual string parsing on non-Windows

### OGL_SetDefaults
- Signature: `void OGL_SetDefaults(OGL_ConfigureData& Data)`
- Purpose: Initialize configuration struct with sensible rendering defaults
- Inputs: Reference to `OGL_ConfigureData` to populate
- Outputs/Return: None (modifies passed struct)
- Side effects: Writes to all fields of `Data`
- Calls: None
- Notes: Sets texture filters (GL_LINEAR for near, varies for far), flags (fader, map, HUD, fog, etc.), default void color (black)

### OGL_TextureOptionsBase::Load
- Signature: `void OGL_TextureOptionsBase::Load()`
- Purpose: Load texture image files (color, normal map, glow, bump offset) from disk into memory
- Inputs: Member file paths (`NormalColors`, `GlowColors`, `NormalMask`, `OffsetMap`)
- Outputs/Return: None (populates member image objects: `NormalImg`, `GlowImg`, `OffsetImg`)
- Side effects: File I/O; allocates image data; queries GPU max texture size
- Calls: `ImageLoader::LoadFromFile()`, `Minify()`, `Clear()` on image objects; `OGL_CheckExtension()`
- Notes: Enforces constraint that glow texture is only present if normal texture exists and dimensions match; respects NPOT flag and mipmapping settings

### OGL_StartProgress / OGL_ProgressCallback / OGL_StopProgress
- Signature: `void OGL_StartProgress(int total_progress)` / `void OGL_ProgressCallback(int delta_progress)` / `void OGL_StopProgress()`
- Purpose: Manage visual progress feedback during long-running resource loads
- Inputs: Total/delta progress counts; callback throttles updates to ~33ms intervals
- Outputs/Return: None
- Side effects: Updates static progress state; renders load screen or dialog; calls `machine_tick_count()` for timing
- Calls: `OGL_LoadScreen::instance()` (preferred) or `open_progress_dialog()` / `draw_progress_bar()` / `close_progress_dialog()` fallbacks
- Notes: Prefers custom load screen texture over generic progress dialog if available

### OGL_LoadModelsImages / OGL_UnloadModelsImages
- Signature: `void OGL_LoadModelsImages(short Collection)` / `void OGL_UnloadModelsImages(short Collection)`
- Purpose: Load or unload all texture and model resources for a game collection (walls, sprites, inhabitants, etc.)
- Inputs: Collection index (0ΓÇô31, see `shape_descriptors.h`)
- Outputs/Return: None
- Side effects: GPU texture/model allocation or deallocation; queries `GL_MAX_TEXTURE_SIZE` and extension support
- Calls: `OGL_LoadTextures()`, `OGL_LoadModels()`, `OGL_UnloadTextures()`, `OGL_UnloadModels()` (defined elsewhere)
- Notes: Checks `OGL_Flag_3D_Models` to conditionally load 3D models; determined elsewhere compiled with/without HAVE_OPENGL

### parse_mml_opengl / reset_mml_opengl
- Signature: `void parse_mml_opengl(const InfoTree& root)` / `void reset_mml_opengl()`
- Purpose: Parse XML/MML configuration for OpenGL settings (textures, models, shaders, fog); reset to defaults
- Inputs: `InfoTree` XML document structure
- Outputs/Return: None (modifies global state: `FogData`, textures, models, shaders)
- Side effects: Allocates/frees `OriginalFogData` backup; modifies global fog configurations
- Calls: `parse_mml_opengl_texture()`, `parse_mml_opengl_model()`, `parse_mml_opengl_shader()`, `InfoTree::children_named()`, `InfoTree::read_attr()`, `InfoTree::read_color()`
- Notes: Backs up original fog data before modification to enable clean resets

### sRGB Color Wrapper Functions (SglColor3f, SglColor4f, etc.)
- Signature: `void SglColor3f(GLfloat r, GLfloat g, GLfloat b)` + variants (3fv, 3ub, 3us, 3usv, 4f, 4fv, 4usv)
- Purpose: Wrap OpenGL color-setting functions to apply sRGB gamma correction when enabled
- Inputs: Color components (float 0ΓÇô1 or unsigned integer) ┬▒ alpha channel
- Outputs/Return: None (direct GL calls)
- Side effects: Calls `glColor*()` with transformed values
- Calls: `sRGB_frob()` (inline conversion), `glColor3fv()`, `glColor4fv()`
- Notes: Comment indicates these belong elsewhere; conversion only applied if `Using_sRGB` flag is set; handles byte (ubyte), short (ushort), and float inputs

## Control Flow Notes
1. **Init phase:** `OGL_Initialize()` detects OpenGL support; if successful, `OGL_SetDefaults()` and `parse_mml_opengl()` configure rendering
2. **Load phase:** `OGL_LoadModelsImages()` called per collection; spawns progress tracking with `OGL_StartProgress()` and `OGL_ProgressCallback()`
3. **Runtime:** sRGB color wrappers called during rendering if color gamma correction is enabled
4. **Shutdown/Reset:** `OGL_UnloadModelsImages()` clears resources; `reset_mml_opengl()` reverts to default configuration

## External Dependencies
- **OpenGL (conditional `HAVE_OPENGL`):** `OGL_Headers.h` (includes GLEW on Windows, SDL2_opengl on Unix), `OGL_Shader.h`
- **File/Resource I/O:** `FileSpecifier`, `ImageLoader` (texture loading), `OGL_LoadScreen` (progress screen)
- **Configuration:** `InfoTree` (XML/MML parsing)
- **Progress UI:** `open_progress_dialog()`, `draw_progress_bar()`, `close_progress_dialog()`, `machine_tick_count()` (defined elsewhere)
- **Game data:** `shape_descriptors.h` (collection/shape constants)
- **Standard library:** `<vector>`, `<string>`, `<math>`
