# Source_Files/RenderMain/AnimatedTextures.cpp
## File Purpose
Implements animated wall textures for the Aleph One game engine. Manages frame sequences that cycle through texture indices at configurable rates, supporting bidirectional animation. Animations are stored per collection and applied during texture lookup via XML configuration.

## Core Responsibilities
- Store and manage animation sequences (frame lists) indexed by collection
- Update animation state (frame and tick phases) each frame
- Translate texture descriptors to their current animated frame
- Parse XML configuration to create and configure animation objects
- Support selective texture animation (animate specific texture or all in list)
- Support bidirectional animation (positive/negative tick rates)
- Clear and reset animation data

## External Dependencies
- **Standard Library**: `<vector>`, `<string.h>`
- **Engine**: `cseries.h` (base types, macros), `interface.h` (shape descriptor macros, `get_number_of_collection_frames()`), `InfoTree.h` (XML parsing)
- **Macros defined elsewhere**: `GET_DESCRIPTOR_SHAPE()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_COLLECTION()`, `GET_COLLECTION_CLUT()`, `BUILD_COLLECTION()`, `BUILD_DESCRIPTOR()`, `UNONE`, `NUMBER_OF_COLLECTIONS`

# Source_Files/RenderMain/AnimatedTextures.h
## File Purpose
Header providing the public interface for animated texture handling in the Aleph One engine. Defines functions to update animated textures each frame, translate texture descriptors to account for animation state, and configure animated textures via XML.

## Core Responsibilities
- Update animated texture state each frame
- Translate shape descriptors to map base textures to their current animated frame
- Parse MML (Marathon Markup Language) XML configuration for animated textures
- Reset animated texture configuration to defaults

## External Dependencies
- **Includes:** `shape_descriptors.h` (defines shape_descriptor typedef and bit-field macros)
- **Forward declarations:** `InfoTree` (defined elsewhere; XML/MML parse tree)
- **Defined elsewhere:** Implementation of all four functions; actual animated texture state/tables

# Source_Files/RenderMain/collection_definition.h
## File Purpose
Defines binary-compatible data structures for Marathon/Aleph One game asset collections. Collections package shapes, animations, bitmaps, and color palettes into discrete files categorized by type (wall, object, interface, scenery).

## Core Responsibilities
- Define `collection_definition` as the root container for all assets in a collection file
- Define shape animation metadata (`high_level_shape_definition`) with frame timing and sound cues
- Define individual sprite properties (`low_level_shape_definition`) including mirroring, origin/key points, and lighting
- Define color palette entry format (`rgb_color_value`) with luminescence flag
- Provide collection type enumeration and file format versioning constants
- Enforce binary layout compatibility with hardcoded struct sizes

## External Dependencies
- `cstypes.h`: provides fixed-width types (`int16`, `uint16`, `int32`, `uint32`, `uint8`, `_fixed`), `FIXED_ONE`, and fixed-point macros
- `<vector>`: STL containers (`std::vector`) used in `collection_definition` for runtime storage of parsed assets
- Forward declarations: `bitmap_definition` (not defined here)


# Source_Files/RenderMain/Crosshairs.h
## File Purpose
Interface header for crosshair rendering and configuration in the game engine. Defines the data structure for crosshair properties (color, thickness, shape, opacity) and provides functions for state management, configuration, and rendering to an SDL surface.

## Core Responsibilities
- Define `CrosshairData` struct encapsulating crosshair visual properties and rendering state
- Provide configuration dialog interface for player customization
- Manage crosshair active/inactive state
- Render crosshairs to the backbuffer
- Expose access to stored crosshair preferences

## External Dependencies
- **cseries.h** ΓÇô Provides `RGBColor` struct (three uint16 components: red, green, blue) and platform abstraction
- **SDL2/SDL.h** ΓÇô Imported indirectly through cseries.h; `SDL_Surface` used as rendering target

# Source_Files/RenderMain/Crosshairs_SDL.cpp
## File Purpose
Implements SDL-based rendering of the game's crosshair HUD element in two configurable shapes. Manages crosshair visibility state and draws crosshairs to an SDL surface each frame, or delegates to Lua HUD if enabled.

## Core Responsibilities
- Maintain crosshair active/inactive state via file-static variable
- Render crosshairs in two shape modes: standard (4 rectangles) or circular (octagon with line segments)
- Map crosshair color data from 16-bit RGB to SDL pixel format
- Calculate crosshair geometry relative to surface center with configurable dimensions
- Check Lua HUD override flag before rendering

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_MapRGB()`, `SDL_FillRect()`
- **Crosshairs.h:** `CrosshairData` struct, `GetCrosshairData()` function
- **screen_drawing.h:** `draw_line()` function (for octagon edges)
- **world.h:** `world_point2d` structure
- **cseries.h:** Base types (`uint32`, `RGBColor`)

# Source_Files/RenderMain/DDS.h
## File Purpose
Defines DirectDraw Surface (DDS) file format structures and flag constants for texture loading. Implements the DDS file format specification from DirectX 9, allowing the engine to parse and read DDS-formatted texture files. Includes conditional guards to avoid conflicts with system DDRAW headers.

## Core Responsibilities
- Define DDS surface descriptor structure (DDSURFACEDESC2) for file header parsing
- Define DDS surface capability flags (DDSCAPS, DDSCAPS2)
- Define pixel format flags (DDPF_*) for texture color/alpha interpretation
- Define surface description flags (DDSD_*) to indicate which fields are valid
- Provide header-only DDS format support without system DDRAW dependency

## External Dependencies
- `cstypes.h` ΓÇö provides `uint32` typedef via SDL2 types
- Conditional: `#ifndef __DDRAW_INCLUDED__` ΓÇö guards against redefinition if system DirectDraw headers are already loaded

# Source_Files/RenderMain/ImageLoader.h
## File Purpose
Defines the `ImageDescriptor` class for holding pixel data and metadata, along with image loading utilities. Supports loading images from files (particularly DDS format), mipmap operations, and format conversions (RGBA8, DXTC compression). Part of the Aleph One game engine's rendering subsystem.

## Core Responsibilities
- **Image data container**: Holds pixel buffer, dimensions, scales, and format information
- **File loading**: Load images from DDS files with optional mipmap support
- **Mipmap management**: Generate, access, and query mipmap levels
- **Format conversion**: Convert between RGBA8 and DXTC compression formats
- **Alpha blending**: Premultiply alpha channel
- **Copy-on-edit pattern**: Template class for lazy-copy resource management
- **Pixel access**: Direct and bulk access to image pixels

## External Dependencies
- **DDS.h**: Direct Draw Surface format structures (`DDSURFACEDESC2`)
- **FileHandler.h**: File abstraction (`FileSpecifier`, `OpenedFile`)
- **cseries.h**: Core utilities and macros
- **\<vector>**: Standard library
- **cstypes.h** (via cseries.h): Fixed-width integer types (`uint32`, `int16`)


# Source_Files/RenderMain/ImageLoader_SDL.cpp
## File Purpose
SDL-based implementation for loading image files into `ImageDescriptor` objects. Converts image data from files (DDS, PNG, BMP, etc.) to 32-bit RGBA surfaces and optionally extracts opacity/grayscale information.

## Core Responsibilities
- Load image files from disk via SDL_image or SDL BMP fallback
- Convert loaded images to platform-correct 32-bit RGBA format (handling endianness)
- Resize images to powers of two if requested
- Process two image modes: color data and opacity/grayscale data
- Extract opacity from grayscale/alpha channels and store as per-pixel alpha
- Delegate DDS format loading to `LoadDDSFromFile()`
- Validate dimension/scale consistency when loading opacity over existing color data

## External Dependencies
- **Includes:** `ImageLoader.h`, `FileHandler.h`, `<SDL2/SDL_image.h>` (conditional), `<cmath>`
- **Symbols defined elsewhere:**
  - `ImageDescriptor::LoadDDSFromFile()`, `ImageDescriptor::Resize()`, `ImageDescriptor::GetPixelBasePtr()` (class methods)
  - `FileSpecifier::Open()` (file abstraction)
  - `OpenedFile::GetRWops()` (SDL RWops accessor)
  - `NextPowerOfTwo()`, `PlatformIsLittleEndian()`, `PIN()` (utility functions/macros)
  - `IMG_Load_RW()`, `SDL_LoadBMP_RW()`, `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`, `SDL_SetSurfaceBlendMode()` (SDL2 API)
  - `temporary`, `csprintf()`, `vassert()` (logging/assertion, defined elsewhere)

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

## External Dependencies
- **AStream.h**: `AIStreamLE` for binary parsing with endianness
- **SDL2**: `SDL_CreateRGBSurfaceFrom`, `SDL_BlitSurface`, `SDL_FreeSurface`, `SDL_SetSurfaceBlendMode`, `SDL_SwapLE16/32` (endian swaps)
- **OpenGL** (conditional): `gluScaleImage` for RGBA8 downscaling; `OGL_IsActive()` gate
- **DDS.h**: `DDSURFACEDESC2` structure and DDSD/DDPF/DDSCAPS flag constants
- **ImageLoader.h**: `ImageDescriptor` class declaration, format enums
- **cstypes.h**: Fixed-size integer types (`uint32`, `uint8`, etc.), `FOUR_CHARS_TO_INT` macro
- **Logging.h**: `logWarning()` macro
- **Undefined (defined elsewhere)**: `PlatformIsLittleEndian()`, `NextPowerOfTwo()`, `OpenedFile` (file abstraction), `FileSpecifier`

# Source_Files/RenderMain/low_level_textures.h
## File Purpose
Low-level software rasterizer for textured polygons in the Aleph One engine. Provides template-based texture mapping for horizontal and vertical screen-aligned polygons with support for multiple pixel formats, blending modes, and effects (tinting, randomization).

## Core Responsibilities
- Template-based texture rasterization for horizontal and vertical polygons
- Multi-mode pixel blending: off (opaque), fast (averaging), and nice (alpha-blended)
- Landscape texture mapping with dynamic width scaling
- Tinting and static/randomization effects for polygons
- Support for 8/16/32-bit pixel formats with transparency handling
- Per-pixel shading table lookup and color channel masking for alpha blending

## External Dependencies
- **SDL2:** `SDL_Surface`, `SDL_PixelFormat`, `SDL_PixelFormat::Rmask/Gmask/Bmask/Rshift/Gshift/Bshift/Rloss/Gloss/Bloss` for pixel format queries
- **cseries.h:** Basic types (`uint16`, `uint32`, `pixel8`, `pixel16`, `pixel32`, `_fixed`, `byte`), constants (`FIXED_FRACTIONAL_BITS`, `TRIG_SHIFT`, `WORLD_FRACTIONAL_BITS`), macros (`MAX`, `MIN`, `fc_assert`)
- **preferences.h:** Blend mode constants (`_sw_alpha_off`, `_sw_alpha_fast`, `_sw_alpha_nice`)
- **textures.h:** `bitmap_definition` structure (width, height, bytes_per_row, row_addresses, flags)
- **scottish_textures.h:** Polygon/tint structures (`_vertical_polygon_data`, `_vertical_polygon_line_data`, `tint_table8/16/32`), constants (`number_of_shading_tables`, `PIXEL8_MAXIMUM_COLORS`, `PIXEL16/32_MAXIMUM_COMPONENT`)

# Source_Files/RenderMain/OGL_Faders.cpp
## File Purpose
Implements OpenGL rendering of fade effects (visual overlays) applied to the game view. Provides functionality to render various fade types including tinting, static/randomization, negation, dodge, burn, and soft-tinting effects to achieve visual feedback for game events.

## Core Responsibilities
- Determine if OpenGL fader rendering is enabled
- Manage access to the fader effect queue
- Apply color transformations (alpha pre-multiplication, color inversion)
- Render up to 6 distinct fade effect types using OpenGL blending modes
- Composite multiple simultaneous faders onto the screen

## External Dependencies
- **OpenGL:** `GL*` functions for state and rendering (blend, color, vertex arrays, logic ops)
- **fades.h:** Fader type constants (`NONE`, `_tint_fader_type`, etc.); `NUMBER_OF_FADER_QUEUE_ENTRIES`
- **Random.h:** `GM_Random` class with `KISS()` and `LFIB4()` methods
- **OGL_Render.h:** `OGL_IsActive()`
- **OGL_Setup.h:** `Get_OGL_ConfigureData()`, flags (`OGL_Flag_Fader`, `OGL_Flag_FlatStatic`)
- **OGL_Headers.h:** OpenGL header inclusion wrapper

# Source_Files/RenderMain/OGL_Faders.h
## File Purpose
Header file for OpenGL screen fade effect rendering in the Aleph One game engine. Defines the interface for managing and rendering fader effects (color overlays with transparency) over the game viewport.

## Core Responsibilities
- Declare whether OpenGL-based faders are currently active
- Define fader queue categories (Liquid, Other) for organizing different fade types
- Define the `OGL_Fader` data structure for fader properties
- Provide queue access to retrieve fader entries by index
- Declare the main fader rendering function that applies fade effects to a rectangular region

## External Dependencies
- `#include "cstypes.h"` ΓÇô provides standard types and the `NONE` constant used in `OGL_Fader` default constructor
- Uses standard C types: `short`, `float`, `bool`

# Source_Files/RenderMain/OGL_FBO.cpp
## File Purpose
Implements OpenGL framebuffer object (FBO) management for off-screen rendering. Provides a single-FBO wrapper (`FBO`) with activation/deactivation stacking, and a dual-FBO ping-ponger (`FBOSwapper`) for post-processing effects like filtering and blending.

## Core Responsibilities
- Create and manage OpenGL framebuffer objects with color textures and depth buffers
- Maintain a stack of active FBOs to support nested rendering contexts
- Implement 2D drawing modes for rendering FBO contents to the screen
- Manage sRGB color-space rendering state during FBO operations
- Provide ping-pong double-buffering between two FBOs for iterative post-processing
- Support filtering, copying, and blending operations between FBOs
- Handle multisample blending with multiple texture units

## External Dependencies
- **OGL_Setup.h:** Global flags (`Using_sRGB`, `Bloom_sRGB`), sRGB extension macros, texture type info (`TxtrTypeInfoList`).
- **OGL_Render.h:** `OGL_RenderTexturedRect()` function for texture drawing.
- **OGL_Textures.h:** Texture info structures (`TxtrTypeInfoData`, `OGL_Txtr_HUD`).
- **OpenGL EXT functions:** `glGenFramebuffersEXT`, `glBindFramebufferEXT`, `glCheckFramebufferStatusEXT`, `glGenRenderbuffersEXT`, etc. (legacy ARB/EXT extensions).

# Source_Files/RenderMain/OGL_FBO.h
## File Purpose
Defines classes for managing OpenGL Framebuffer Objects (FBOs) used for off-screen rendering and texture rendering targets. Provides utilities for activating, deactivating, and compositing between FBOs, supporting both single FBO and double-buffered FBO swapping patterns.

## Core Responsibilities
- Manage individual FBO lifecycle (allocation, activation, deactivation, cleanup)
- Track active FBOs via a static chain for nested/stacked activation
- Provide drawing and composition operations (blit, blend, filter, copy)
- Implement FBOSwapper for ping-pong rendering (e.g., post-processing pipelines)
- Handle sRGB color space management for FBOs
- Support clearing and state management for FBO render targets

## External Dependencies
- **`OGL_Headers.h`**: Provides OpenGL type definitions and extension headers (GLuint, GL_FRAMEBUFFER_EXT, GLEW/SDL_opengl)
- **`cseries.h`**: Aleph One utility headers (types, macros)
- **`<vector>`** (STL): Dynamic array for `active_chain`
- **Defined elsewhere**: Actual method implementations, OpenGL binding/state calls, texture setup

# Source_Files/RenderMain/OGL_Headers.h
## File Purpose
Cross-platform compatibility header that conditionally includes OpenGL headers based on the target platform. Provides a single include point for all Aleph One rendering code to access OpenGL without managing platform-specific header differences.

## Core Responsibilities
- Guard against multiple inclusion and provide consistent OpenGL access points
- Abstract platform-specific OpenGL header paths (Windows/Unix/macOS)
- Enable static GLEW linking on Windows (`GLEW_STATIC`)
- Define `GL_GLEXT_PROTOTYPES` for Unix/Linux platforms
- Check for OpenGL support via `HAVE_OPENGL` feature detection

## External Dependencies
- **Windows:** `<GL/glew.h>` (static linking via `GLEW_STATIC` macro)
- **Unix/Linux:** `<SDL2/SDL_opengl.h>` with `GL_GLEXT_PROTOTYPES` enabled
- **macOS:** `<OpenGL/glu.h>` for GLU utilities
- **Linux fallback:** `<GL/glu.h>` (when not on macOS)
- **Project config:** `"config.h"` (feature detection: `HAVE_OPENGL`, `__WIN32__`, `__APPLE__`, `__MACH__`)

# Source_Files/RenderMain/OGL_Model_Def.cpp
## File Purpose
Manages OpenGL 3D model loading, storage, and runtime access for the Aleph One engine. Handles model format conversions, geometric transformations, skin/texture association, and MML configuration parsing for Marathon's model system.

## Core Responsibilities
- Load 3D models from multiple file formats (Wavefront OBJ, 3D Studio Max, Dim3, QuickDraw 3D)
- Map Marathon-engine animation sequences to model sequences via hash tables
- Apply geometric transformations (rotation, scaling, shifting) to loaded geometry
- Store and retrieve models per collection with fast O(1) hashing
- Manage model skins (textures) and associated OpenGL texture IDs
- Parse XML (MML) model configuration and register into model database
- Unload models and release OpenGL resources on demand

## External Dependencies
- **Includes:** `cseries.h` (Marathon types), `OGL_Setup.h`, `OGL_Model_Def.h`, `Dim3_Loader.h`, `StudioLoader.h`, `WavefrontLoader.h`, `InfoTree.h`
- **External functions:** `LoadModel_Wavefront`, `LoadModel_Studio`, `LoadModel_Dim3`, `LoadModel_QD3D` (model loaders); `OGL_ProgressCallback`, `Get_OGL_ConfigureData` (OpenGL setup)
- **External types:** `FileSpecifier`, `Model3D`, `InfoTree`, `OGL_SkinData`, OpenGL (GL*).
- **Defined elsewhere:** `NONE`, `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`, `ALL_CLUTS`, `INFRAVISION_BITMAP_SET`, `SILHOUETTE_BITMAP_SET`, infravision/silhouette helper predicates.

# Source_Files/RenderMain/OGL_Model_Def.h
## File Purpose
Defines OpenGL model and skin management structures for rendering 3D models with multiple texture variants and transformation options. Provides declarations for loading/unloading models, managing skins (per-CLUT texture variants), and parsing data-driven model configuration from MML files.

## Core Responsibilities
- Define `OGL_SkinData` and `OGL_SkinManager` for managing per-CLUT texture variants with opacity/blending options
- Define `OGL_ModelData` as primary model configuration container with transformation, lighting, and depth settings
- Provide lookup functions to retrieve model data by game collection/sequence ID
- Manage collection-level model loading/unloading and texture resource initialization
- Parse and apply Marathon Mark-up Language (MML) model definitions
- Support multiple lighting modes and normal/depth rendering options
- Manage sprite depth-sorting override state

## External Dependencies
- `OGL_Texture_Def.h`: `OGL_TextureOptionsBase` (base class), texture enums, `MAXIMUM_CLUTS_PER_COLLECTION`, `NUMBER_OF_OPENGL_BITMAP_SETS`
- `Model3D.h`: `Model3D` struct (geometry storage)
- `OGL_Headers.h`: OpenGL types/functions (`GLuint`, `GLushort`, etc.)
- STL: `vector<>`
- Elsewhere: `FileSpecifier`, `ImageDescriptor`, `InfoTree`

# Source_Files/RenderMain/OGL_Render.cpp
## File Purpose
OpenGL rendering backend for the Aleph One game engine (Marathon source port). Implements coordinate transformation pipelines, frame setup/teardown, matrix management, texture preloading, 3D model rendering with lighting/shading, and 2D UI rendering (crosshairs, text, debug geometry).

## Core Responsibilities
- OpenGL context lifecycle (initialization, frame start/end, shutdown)
- Coordinate system transformations (Marathon world ΓåÆ OpenGL eye ΓåÆ screen)
- Projection matrix selection and management
- Texture preloading to avoid lazy-loading stalls
- 3D sprite/model rendering with per-vertex lighting and transfer modes
- Fog configuration and rendering
- Static effect rendering (stipple or stencil-based noise)
- 2D UI overlay rendering (crosshairs, text, geometric primitives)
- Shader setup and callback dispatch

## External Dependencies
- **OpenGL:** `OGL_Headers.h` (GL/GLEW headers); `glMatrixMode`, `glLoadMatrixd`, `glEnable`, `glDisable`, `glFogf`, `glClipPlane`, etc.
- **Texture management:** `OGL_Textures.h`, `AnimatedTextures.h` (TextureManager, AnimTxtr_Translate)
- **Model rendering:** `ModelRenderer.h`, `OGL_Shader.h` (ModelRenderShader, shader callbacks)
- **Game world:** `world.h`, `map.h`, `player.h`, `render.h` (polygon_data, dynamic_world, local_player)
- **Preferences:** `preferences.h`, `OGL_Setup.h` (graphics_preferences, OGL configuration)
- **UI/Debug:** `Crosshairs.h`, `OGL_Faders.h`, `ViewControl.h`, `Logging.h`
- **Math:** `VecOps.h`, `Random.h` (vector operations, GM_Random)
- **Display:** `screen.h`, `interface.h` (MainScreenIsOpenGL, GetOnScreenFont)

**Notable undefined symbols:** `get_shape_bitmap_and_shading_table`, `FindShadingColor`, `normalize_angle`, `cosine_table`, `sine_table`, `WORLD_ONE`, `FIXED_ONE`, `OGL_BlendType_*` constants.

# Source_Files/RenderMain/OGL_Render.h
## File Purpose
OpenGL interface header providing functions to integrate OpenGL 3D rendering with the Marathon (Aleph One) game engine. Declares the public API for managing rendering context, setting view parameters, and rendering geometric primitives, sprites, text, and UI elements.

## Core Responsibilities
- **Context lifecycle**: Initialize/destroy OpenGL rendering context and manage GPU resources
- **View configuration**: Set perspective parameters, camera position, and projection matrices
- **Geometric rendering**: Render walls, sprites, and other game objects
- **UI rendering**: Display text, crosshairs, cursors, rectangles, and overhead map elements
- **State management**: Query rendering status (active, 2D enabled, current fog) and control rendering modes
- **Rendering modes**: Support foreground (weapons-in-hand) and main view rendering with separate view parameters
- **Buffer management**: Handle window bounds, back buffer allocation, and buffer swapping

## External Dependencies
- **OGL_Setup.h** ΓÇô OpenGL configuration, `OGL_FogData`, color types
- **render.h** ΓÇô `view_data`, `polygon_definition`, `rectangle_definition`
- Implicit: SDL (SDL_Rect), platform rectangles (Rect)
- Implicit: OpenGL headers (called from .cpp implementation, not visible here)

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

## External Dependencies
- **OpenGL (conditional `HAVE_OPENGL`):** `OGL_Headers.h` (includes GLEW on Windows, SDL2_opengl on Unix), `OGL_Shader.h`
- **File/Resource I/O:** `FileSpecifier`, `ImageLoader` (texture loading), `OGL_LoadScreen` (progress screen)
- **Configuration:** `InfoTree` (XML/MML parsing)
- **Progress UI:** `open_progress_dialog()`, `draw_progress_bar()`, `close_progress_dialog()`, `machine_tick_count()` (defined elsewhere)
- **Game data:** `shape_descriptors.h` (collection/shape constants)
- **Standard library:** `<vector>`, `<string>`, `<math>`

# Source_Files/RenderMain/OGL_Setup.h
## File Purpose
Header file defining OpenGL initialization, detection, configuration, and resource management for the Aleph One game engine. Provides interfaces for detecting OpenGL presence, configuring rendering parameters (textures, models, fog), and managing texture/model loading and sRGB color space conversions.

## Core Responsibilities
- OpenGL presence detection and initialization
- Configuration management (texture filtering, color depth, rendering flags, fog, anisotropy)
- Texture and 3D model resource loading/unloading with per-type quality degradation
- Progress tracking for long-running operations
- sRGB color space management (linear Γåö sRGB conversion)
- Fog configuration and rendering modes
- MML (modding markup language) parsing for extensibility

## External Dependencies
- **Included headers:**
  - `OGL_Subst_Texture_Def.h` ΓÇô texture option structures
  - `OGL_Model_Def.h` ΓÇô 3D model and skin definitions
  - `<cmath>` ΓÇô `std::pow()` for sRGB gamma correction
  - `<string>` ΓÇô `std::string` for extension name checking
  
- **Defined elsewhere:**
  - OpenGL types and functions (implicit; guarded by `HAVE_OPENGL`)
  - `Model3D` class (from `Model3D.h`, bundled in model def header)
  - `InfoTree` class (MML parsing infrastructure)
  - `rgb_color`, `RGBColor` types (color definitions)
  - `FileSpecifier` type (file handling)

# Source_Files/RenderMain/OGL_Shader.cpp
## File Purpose
Implements OpenGL shader compilation and lifecycle management for the Aleph One game engine. Provides functionality to load, compile, link, and manage GLSL vertex/fragment shader programs, including fallback error shaders and support for conditional compilation based on hardware capabilities (sRGB, bloom, etc.).

## Core Responsibilities
- Compile GLSL vertex and fragment shaders from source strings with platform-specific preprocessor directives
- Create and manage OpenGL shader programs (creation, linking, cleanup)
- Maintain a global registry of compiled shader programs indexed by type
- Set uniform variables (floats, matrices, textures) in active shaders with caching optimization
- Parse shader definitions from MML/XML configuration files and dynamically reload them
- Provide fallback error shaders when compilation fails
- Support built-in shader sources and load shader sources from disk files

## External Dependencies
- **STL:** `<algorithm>` (std::fill_n), `<iostream>` (fprintf), `<string>`, `<map>`
- **File I/O:** FileHandler.h (FileSpecifier, OpenedFile)
- **OpenGL ARB:** glCreateShaderObjectARB, glShaderSourceARB, glCompileShaderARB, glCreateProgramObjectARB, glAttachObjectARB, glLinkProgramARB, glUseProgramObjectARB, glGetObjectParameterivARB, glGetShaderiv, glGetProgramiv, glGetUniformLocationARB, glGetShaderInfoLog, glGetProgramInfoLog, glDeleteObjectARB, glDeleteProgram, glUniform1iARB, glUniform1fARB, glUniformMatrix4fvARB
- **OGL Configuration:** OGL_Setup.h (global flags: Wanting_sRGB, Bloom_sRGB, DisableClipVertex())
- **Configuration parsing:** InfoTree.h (XML/MML tree abstraction)
- **Logging:** Logging.h (logError macro)

# Source_Files/RenderMain/OGL_Shader.h
## File Purpose
Defines the `Shader` class for managing OpenGL vertex/fragment shader compilation, linking, and runtime state. Provides static factory methods to access pre-loaded shader instances and uniform variable binding for rendering pipeline integration.

## Core Responsibilities
- Compile and link OpenGL shader programs from vertex/fragment source files
- Manage uniform variable locations and cached values for per-frame updates
- Support multiple shader types (landscape, sprite, wall, bloom effects, infravision, etc.)
- Provide global shader instance access and lifecycle management (load all / unload all)
- Enable/disable shaders and set uniform float and matrix values during rendering
- Support multi-pass rendering through `passes()` member

## External Dependencies
- **OGL_Headers.h** ΓÇö `GLhandleARB`, OpenGL function declarations (glGetUniformLocationARB, etc.)
- **FileHandler.h** ΓÇö `FileSpecifier` class for file I/O
- **std::string, std::map** ΓÇö Standard library
- **Friend classes:** `XML_ShaderParser`, `Shader_MML_Parser` (defined elsewhere; parsers for shader definitions)
- **Global functions:** `parse_mml_opengl_shader(const InfoTree&)`, `reset_mml_opengl_shader()` ΓÇö MML-based shader configuration (defined elsewhere; `InfoTree` is forward-declared)

# Source_Files/RenderMain/OGL_Subst_Texture_Def.cpp
## File Purpose
Manages OpenGL substitute texture configurations for walls and sprites in the Aleph One game engine. Stores texture rendering options by collection, loads/unloads GPU textures, and parses MML configuration files to define texture properties.

## Core Responsibilities
- Store and retrieve texture options indexed by (CLUT, Bitmap) pairs across multiple collections
- Provide cascading fallback lookup (exact CLUT ΓåÆ infravision variant ΓåÆ silhouette variant ΓåÆ ALL_CLUTS ΓåÆ default)
- Load and unload GPU textures with progress reporting
- Parse MML configuration to define texture properties (opacity, blending, bloom, images, masks)
- Handle legacy CLUT options and translate to newer variant system
- Support texture variants (normal, infravision, silhouette) with per-CLUT or all-CLUT specialization

## External Dependencies
- `cseries.h`: Platform abstraction, type definitions (int16, short)
- `OGL_Subst_Texture_Def.h`: Module header; `OGL_TextureOptions` struct definition
- `Logging.h`: Logging macros (included but unused here)
- `InfoTree.h`: XML/INI config parser for MML loading
- `<boost/unordered_map.hpp>`: Hash map container
- `OGL_ProgressCallback(int)` ΓÇö defined elsewhere; progress reporting
- `IsInfravisionTable()`, `IsSilhouetteTable()` ΓÇö defined elsewhere; CLUT type checks
- `OGL_TextureOptions::Load()`, `Unload()` ΓÇö defined in OGL_Texture_Def.h

# Source_Files/RenderMain/OGL_Subst_Texture_Def.h
## File Purpose
Header file defining substitute texture configuration and management for OpenGL-rendered walls and sprites in the Aleph One game engine. Extends base texture definitions with sprite-specific billboard modes and texture loading/unloading infrastructure.

## Core Responsibilities
- Define `OGL_TextureOptions` structure for wall/sprite substitute textures
- Provide billboard type enumeration for sprite rendering modes
- Declare texture collection management functions (load/unload/count)
- Declare MML (configuration) parsing and reset functions for texture definitions
- Query current texture options by collection, CLUT, and bitmap ID

## External Dependencies
- `OGL_Texture_Def.h` ΓÇö Base texture options and enums (opacity types, blend types, bitmap set constants)
- `InfoTree` ΓÇö Forward declared; used in MML parsing (defined elsewhere)
- Preprocessor: `HAVE_OPENGL` guard (OpenGL support conditional)

# Source_Files/RenderMain/OGL_Texture_Def.h
## File Purpose
Defines OpenGL texture configuration structures and constants for wall/sprite texture substitutions and model skins in the Aleph One engine. Handles multiple CLUT (color lookup table) variants, opacity modes, blend types, and special effects like infravision/silhouette rendering.

## Core Responsibilities
- Define bitmap set enumeration for color tables, infravision, and silhouette variants
- Enumerate CLUT variants (normal, infravision, silhouette)
- Define opacity types (Crisp, Flat, Avg, Max) and alpha blending modes
- Provide helper functions to identify special CLUT types
- Define `OGL_TextureOptionsBase` struct for texture configuration with opacity, blending, and bloom controls
- Manage image loading with FileSpecifier paths and ImageDescriptor objects

## External Dependencies
- `shape_descriptors.h` ΓÇö provides `MAXIMUM_CLUTS_PER_COLLECTION` macro
- `ImageLoader.h` ΓÇö provides `ImageDescriptor` class and `FileSpecifier` type
- `<vector>` ΓÇö STL vector (included but not directly used in this header)
- `HAVE_OPENGL` ΓÇö preprocessor guard

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

## External Dependencies
- **OpenGL:** `glGenTextures`, `glBindTexture`, `glDeleteTextures`, `glTexImage2D`, `glTexParameteri`, `glTexEnvi`, `gluBuild2DMipmaps`, `glCompressedTexImage2DARB`, `glGetIntegerv`
- **SDL:** `SDL_SwapLE16`, `SDL_endian.h`
- **Marathon engine:** shape descriptors, collection system (`get_bitmap_index`, `get_collection_colors`), map data, preferences
- **Image management:** `ImageDescriptor`, `ImageDescriptorManager`, `ImageLoader` (implied via includes)
- **Engine infrastructure:** `OGL_Setup.h`, `OGL_Render.h`, `OGL_Blitter.h`, `OGL_Headers.h` (OpenGL abstraction)

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

## External Dependencies
- **OGL_Headers.h**: OpenGL API bindings (glBindTexture, GLuint, GLenum, GLdouble)
- **OGL_Subst_Texture_Def.h**: OGL_TextureOptions, BillboardType, texture option parsing
- **scottish_textures.h**: Transfer modes, shading table definitions, shape_descriptor, polygon/rectangle structures
- **ImageDescriptorManager**: Image data wrapper with premultiplication and descriptor queries (defined elsewhere)
- **OGL_TextureOptions** (base class OGL_TextureOptionsBase): Opacity type, blending modes, bloom parameters

# Source_Files/RenderMain/Rasterizer.h
## File Purpose
Abstract base class defining the interface for rasterizer implementations (software, OpenGL, etc.). Establishes the contract that all rasterizer subclasses must implement to handle view setup, polygon/rectangle rendering, and foreground object display.

## Core Responsibilities
- Provide virtual interface for setting view and rendering parameters
- Define entry/exit points for rendering passes (`Begin`, `End`)
- Specify rendering methods for textured polygons (horizontal and vertical)
- Specify rendering method for textured rectangles
- Support foreground object rendering (weapons, HUD elements in first-person)
- Manage horizontal reflection for foreground objects

## External Dependencies
- **render.h** ΓÇö provides `view_data`, `polygon_definition`, `rectangle_definition` type definitions
- **OGL_Render.h** ΓÇö conditionally included (requires `HAVE_OPENGL`); OpenGL-specific declarations
- Standard C++ class and virtual method mechanism

# Source_Files/RenderMain/Rasterizer_OGL.h
## File Purpose
OpenGL-specific implementation of the abstract `RasterizerClass` interface. Provides a thin adapter layer that delegates rendering operations to underlying OpenGL functions. Serves as a pluggable backend for the engine's rendering pipeline when `HAVE_OPENGL` is defined.

## Core Responsibilities
- Implement the rasterizer interface for OpenGL-based rendering
- Manage view setup and camera configuration for OpenGL
- Handle foreground layer rendering (weapons, hand-held objects, HUD elements)
- Delegate polygon (wall) and sprite rendering to OpenGL subsystem functions
- Act as a concrete factory/bridge between the abstract render interface and OGL_Render functions

## External Dependencies
- **Includes:** `Rasterizer.h` (base class definition)
- **Imported functions (from OGL_Render module):** `OGL_SetView()`, `OGL_SetForeground()`, `OGL_SetForegroundView()`, `OGL_StartMain()`, `OGL_EndMain()`, `OGL_RenderWall()`, `OGL_RenderSprite()`
- **External types used:** `view_data`, `polygon_definition`, `rectangle_definition` (defined elsewhere in render headers)
- **Conditional compilation:** Only available if `HAVE_OPENGL` is defined; allows software or alternative rasterizers to coexist

# Source_Files/RenderMain/Rasterizer_Shader.cpp
## File Purpose
Implements shader-based OpenGL rasterization for Aleph One, managing camera transformations, projection matrices, framebuffer composition, and post-processing effects like gamma correction.

## Core Responsibilities
- Configure projection and modelview matrices from view parameters (FOV, position, orientation)
- Compute landscape shader transformation for proper texture alignment
- Manage framebuffer swapping for offscreen rendering and composition
- Apply gamma correction and void-smearing effects during frame output
- Handle view distortion during teleport effects

## External Dependencies

- **Notable includes:**
  - `OGL_Headers.h` ΓÇö OpenGL context/platform abstraction
  - `Rasterizer_Shader.h` ΓÇö Class definition
  - `OGL_Shader.h` ΓÇö Shader class and uniform-setting interface
  - `OGL_FBO.h` ΓÇö FBOSwapper framebuffer management
  - `lightsource.h`, `media.h`, `player.h`, `weapons.h` ΓÇö Game world data (included but not directly used in this file)
  - `preferences.h` ΓÇö gamma_preferences global
  - `screen.h` ΓÇö OGL_RenderFrame, SetForeground functions

- **Defined elsewhere:**
  - `Rasterizer_OGL_Class` ΓÇö Parent class (SetView, Begin, End overridden)
  - `OGL_SetView()` ΓÇö Platform-specific view setup
  - `View_FOV_FixHorizontalNotVertical()` ΓÇö Preference query
  - `MainScreenPixelScale()` ΓÇö DPI/resolution scaling factor
  - `get_actual_gamma_adjust()` ΓÇö Compute gamma multiplier from preferences
  - `Shader` class ΓÇö Uniform management, enablement
  - `FBOSwapper` ΓÇö Framebuffer swapping/composition

# Source_Files/RenderMain/Rasterizer_Shader.h
## File Purpose
Defines `Rasterizer_Shader_Class`, a shader-based OpenGL rasterizer for the Aleph One game engine. Extends the base OpenGL rasterizer with advanced rendering techniques including frame buffer object (FBO) swapping and void-smearing effects.

## Core Responsibilities
- Provide shader-based rendering interface extending `Rasterizer_OGL_Class`
- Manage frame buffer object (FBO) swapping for efficient render-target handling
- Configure and initialize OpenGL state for shader-based rendering
- Bracket rendering operations with Begin/End lifecycle methods
- Support visual effects like smearing void regions
- Track viewport dimensions (width/height) for render target management

## External Dependencies
- **Rasterizer_OGL.h**: Base class `Rasterizer_OGL_Class` providing core OpenGL rendering interface
- **cseries.h**: Cross-platform utilities (types, macros)
- **map.h**: World/map data structures (used by view_data)
- **std::memory**: `std::unique_ptr<FBOSwapper>` for RAII resource management
- **FBOSwapper** (forward declared): Frame buffer object swapper; definition elsewhere
- **view_data** (from map.h or related): Camera view configuration structure
- **RenderRasterize_Shader** (friend class): Privileged access to shader rasterizer internals

Compilation guarded by `#ifdef HAVE_OPENGL` ΓÇö not available if OpenGL support is disabled.

# Source_Files/RenderMain/Rasterizer_SW.h
## File Purpose
Software-based rasterizer implementation for the game engine's rendering pipeline. Provides concrete texture rendering methods for horizontal/vertical polygons and rectangles via the Scottish Textures system. Inherits from the abstract `RasterizerClass` to support multiple rendering backends.

## Core Responsibilities
- Implement software-based polygon and rectangle rasterization
- Manage view/camera data and screen framebuffer pointers
- Provide entry points for the Scottish Textures rendering system
- Support both horizontal and vertical polygon orientations
- Maintain compatibility with the abstract rasterizer interface

## External Dependencies
- **Base class**: `RasterizerClass` (Rasterizer.h) ΓÇö defines virtual interface for swappable rasterizer backends
- **Type definitions** (defined elsewhere): `view_data`, `bitmap_definition`, `polygon_definition`, `rectangle_definition` ΓÇö likely in render.h or related headers
- **Implementation**: scottish_textures.c ΓÇö contains actual rasterization algorithms (not visible in this header)

# Source_Files/RenderMain/render.cpp
## File Purpose

This is the central orchestration module for the rendering pipeline. It initializes and updates camera/view state each frame, coordinates visibility determination, polygon sorting, object placement, and rasterization into a coherent render pass, handles visual effects (explosions, teleports, camera shakes), applies transfer modes (texture slides, wobbles, pulsates, fades), and renders the weapon/HUD layer. It also supports Marathon 1 exploration missions by checking unseen polygons in the background.

## Core Responsibilities

- Initialize view/camera data with perspective calculations (FOV, cone angles, screen-to-world scaling)
- Update view state each tick (yaw/pitch vectors, FOV transitions, vertical pitch scaling)
- Apply render effects (fold in/out teleports, explosion camera shake) and transfer mode animations
- Orchestrate the multi-stage rendering pipeline: visibility tree ΓåÆ polygon sorting ΓåÆ object placement ΓåÆ rasterization
- Route rendering to appropriate rasterizer backend (software, OpenGL classic, or shader-based)
- Render player weapons and HUD sprites in foreground
- Manage transfer modes for both polygons (floors, ceilings, walls) and sprites/rectangles with various effects (slides, wobbles, static, fades)
- Support Marathon 1 exploration missions by periodically checking player views against unexplored polygons
- Allocate and initialize render memory and subsystem objects

## External Dependencies

- **Notable includes:**
  - `map.h` ΓÇô polygon, line, endpoint, side structures; map accessors
  - `render.h` ΓÇô view_data, render flag definitions
  - `lightsource.h` ΓÇô light intensity lookups
  - `media.h` ΓÇô media/liquid state
  - `weapons.h` ΓÇô weapon display info
  - `player.h` ΓÇô player location and data
  - `RenderVisTree.h`, `RenderSortPoly.h`, `RenderPlaceObjs.h`, `RenderRasterize.h` ΓÇô render subsystem classes
  - `Rasterizer_SW.h`, `Rasterizer_OGL.h`, `Rasterizer_Shader.h` ΓÇô rasterizer backends
  - `OGL_Render.h` ΓÇô OpenGL-specific rendering (conditional)
  - `dynamic_limits.h`, `AnimatedTextures.h`, `preferences.h`, `screen.h` ΓÇô configuration and utility

- **External symbols used:**
  - `map_polygons`, `map_endpoints`, `map_lines` ΓÇô map geometry (from map.h)
  - `players`, `dynamic_world` ΓÇô game state (from player.h, map.h)
  - `graphics_preferences` ΓÇô renderer backend selection (preferences)
  - `render_computer_interface()`, `render_overhead_map()` ΓÇô external renderers (screen.c)
  - `get_weapon_display_information()`, `extended_get_shape_information()`, `extended_get_shape_bitmap_and_shading_table()` ΓÇô shape/texture accessors (interface.h)
  - `OGL_GetModelData()` ΓÇô 3D model lookup (OGL_Render.h)
  - Trig tables: `cosine_table[]`, `sine_table[]`, `NORMALIZE_ANGLE()` ΓÇô fixed-point math

# Source_Files/RenderMain/render.h
## File Purpose
Central render system header defining the camera/view state structure (`view_data`), visibility/render flags for world geometry, and function prototypes for rendering initialization, frame rendering, and visual effects. Serves as the interface between the game engine and rendering backends (software and OpenGL).

## Core Responsibilities
- Define the `view_data` structure encapsulating all view parameters: camera position, orientation, FOV, screen dimensions, and effect state
- Declare render flag macros and bit definitions for marking polygon/endpoint/side visibility during traversal
- Export memory allocation and initialization for the rendering system
- Provide entry points for main view rendering, effect triggering, overhead map, and UI rendering
- Define render effects (fold-in/out, explosion) and shading modes (normal, infravision)

## External Dependencies
- **world.h**: `world_point3d`, `world_distance`, `angle`, `world_vector3d`, `fixed_angle`, trigonometric types
- **textures.h**: `bitmap_definition` (texture/sprite data)
- **scottish_textures.h**: `rectangle_definition` (sprites), `polygon_definition` (walls/floors/ceilings)
- **ViewControl.h**: `View_FOV_Normal()`, `View_FOV_ExtraVision()`, `View_FOV_TunnelVision()` (FOV accessors); effect and effect-phase control functions
- Implicit OpenGL context (for rendering backend; not included in this header)

# Source_Files/RenderMain/RenderPlaceObjs.cpp
## File Purpose
Implements object placement and depth-sorting for rendering. Converts game objects into renderer-ready data structures with proper projection, visibility culling, clipping windows, and depth ordering within a polygon tree.

## Core Responsibilities
- Build sorted render-object list from world objects per polygon
- Create render objects with projected screen coordinates and shape/transfer-mode data
- Determine which polygons (nodes) an object spans via line-crossing analysis
- Sort objects into depth tree based on intersection testing
- Aggregate clipping windows from multiple nodes
- Handle 3D model bounding-box projection and scaling
- Support parasitic objects (objects attached to host objects)

## External Dependencies
- `map.h`, `world.h` ΓÇö polygon/object/light structs, accessor functions
- `lightsource.h` ΓÇö light intensity lookup
- `media.h` ΓÇö media height queries
- `render.h`, `RenderSortPoly.h` ΓÇö render tree and sorted-node structures
- `OGL_Setup.h` ΓÇö 3D model data and queries
- `ChaseCam.h` ΓÇö chase-cam opacity
- `player.h` ΓÇö current player index
- `ephemera.h` ΓÇö ephemera object queries
- `preferences.h` ΓÇö graphics preferences (ephemera quality)
- STL: `<vector>`, `<algorithm>`, `<boost/container/small_vector.hpp>`

# Source_Files/RenderMain/RenderPlaceObjs.h
## File Purpose
Defines a class that organizes game objects (inhabitants) into a visibility-sorted tree for correct rendering order. Acts as a bridge between game objects and the rendering subsystem, managing object placement, clipping, and depth sorting. Part of the Aleph One game engine rendering pipeline.

## Core Responsibilities
- Build render objects from game objects with lighting and opacity parameters
- Sort render objects into a visibility tree (polygon-based spatial structure)
- Calculate and manage per-object clipping windows for screen rendering
- Maintain linked lists of objects per sorted polygon node (interior/exterior)
- Integrate with visibility tree and polygon sorting systems
- Rescale shape information as needed for rendering

## External Dependencies
- Includes: `<vector>`, `world.h`, `interface.h`, `render.h`, `RenderSortPoly.h`
- Uses: `view_data` (view parameters), `sorted_node_data` (polygon nodes), `clipping_window_data` (screen clipping), `object_data` (game objects), `shape_information_data` (sprite metrics), `RenderVisTreeClass`, `RenderSortPolyClass` (visibility/sorting results)

# Source_Files/RenderMain/RenderRasterize.cpp
## File Purpose
Implements polygon clipping and rasterization for the Aleph One game engine. Processes a sorted tree of world geometry and clips polygons to viewport boundaries, converting them to screen-space textured polygons for rendering. Handles complex cases like semi-transparent liquids, flooded platforms, and animated textures.

## Core Responsibilities
- Tree traversal: iterates sorted polygons from back-to-front
- Polygon clipping: clips horizontal and vertical polygons to viewport boundaries using Sutherland-Hodgman algorithm
- Coordinate transformation: converts world coordinates to screen-space with perspective division
- Surface rendering: dispatches floors, ceilings, walls, and objects to rasterizer
- Liquid media handling: manages rendering of see-through vs opaque liquid surfaces
- Texture animation translation: applies animated texture frame selection
- Void detection: tracks whether polygon boundaries face empty space (for transparency rendering)

## External Dependencies
- **Game world data:** `polygon_data`, `side_data`, `line_data`, `endpoint_data`, `platform_data` (map.h); `media_data` (media.h); `light_data`, `get_light_intensity()` (lightsource.h)
- **Rendering:** `RasterizerClass`, `RenderSortPolyClass`, `polygon_definition`, `view_data` (render.h, Rasterizer.h, RenderSortPoly.h)
- **Textures:** `AnimTxtr_Translate()` (AnimatedTextures.h); `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()` (render.h)
- **Configuration:** `OGL_ConfigureData`, `Get_OGL_ConfigureData()`, `OGL_Flag_LiqSeeThru` (OGL_Setup.h); `graphics_preferences`, `screen_mode` (preferences.h, screen.h)
- **Platform:** `PLATFORM_IS_FLOODED()`, `find_flooding_polygon()` (platforms.h, map.h)

# Source_Files/RenderMain/RenderRasterize.h
## File Purpose
Defines RenderRasterizerClass, which converts a sorted visibility tree and placed objects into screen-space geometry for rasterization. Acts as the bridge between world-space rendering data and platform-specific rasterizers (software or OpenGL), handling clipping, lighting, and coordinate transforms.

## Core Responsibilities
- Traverse sorted polygon tree (RenderSortPolyClass) in depth order
- Rasterize floors, ceilings, and walls with visibility clipping windows
- Render sprite objects with proper depth ordering and media-boundary semantics
- Apply geometric clipping (XY screen bounds, Z elevation, XZ vertical bounds)
- Delegate final pixel output to RasterizerClass backend
- Support multi-pass rendering (diffuse + glow layers)

## External Dependencies
- `<vector>` ΓÇö STL
- `world.h` ΓÇö long_vector2d, world_distance, angle, long_point3d
- `render.h` ΓÇö view_data, polygon_data, bitmap_definition, render flags
- `RenderSortPoly.h` ΓÇö sorted_node_data, RenderSortPolyClass
- `RenderPlaceObjs.h` ΓÇö render_object_data
- `Rasterizer.h` ΓÇö RasterizerClass (abstract base for platform-specific rendering)
- **Defined elsewhere**: endpoint_data, horizontal_surface_data, clipping_window_data, line_clip_data, rectangle_definition, side_texture_definition

# Source_Files/RenderMain/RenderRasterize_Shader.cpp
## File Purpose
Implements shader-based rasterization rendering for Aleph One, extending the base RenderRasterizerClass to handle OpenGL rendering of world geometry, sprites, and effects with support for glow/bloom postprocessing, infravision, and multiple transfer/blending modes.

## Core Responsibilities
- Initialize OpenGL shaders and bloom/blur postprocessing pipeline
- Execute multi-pass rendering (diffuse and glow passes) with dynamic shader selection
- Configure shader uniforms for view parameters, lighting, fog, and animation
- Render world polygons (floors, ceilings, walls) with texture coordinates and animations
- Render game objects (sprites and 3D models) with proper depth handling and transparency
- Render screen-space HUD/weapon sprites with perspective clipping
- Manage viewport clipping planes and camera-relative transforms

## External Dependencies
- **OpenGL**: `OGL_Headers.h` (GLEW/SDL_opengl), `OGL_Shader.h`, `OGL_FBO.h`, `OGL_Textures.h`
- **Game world**: `lightsource.h`, `media.h`, `player.h`, `weapons.h`, `AnimatedTextures.h`, `ChaseCam.h`, `preferences.h`
- **Base classes**: `RenderRasterize.h` (RenderRasterizerClass), `Rasterizer_Shader.h`
- **External functions** (defined elsewhere): `get_endpoint_data()`, `get_polygon_data()`, `get_media_data()`, `get_light_intensity()`, `View_GetLandscapeOptions()`, `OGL_GetCurrFogData()`, `AnimTxtr_Translate()`, `FindInfravisionVersionRGBA()`, `get_weapon_display_information()`, `LoadModelSkin()`, `FlatBumpTexture()`
- **Globals**: `current_player`, `view`, `cosine_table[]`, `sine_table[]`

# Source_Files/RenderMain/RenderRasterize_Shader.h
## File Purpose
Defines a shader-based GPU rendering rasterizer that extends the base RenderRasterizerClass to support modern OpenGL rendering with framebuffer objects, shader pipelines, and advanced visual effects. Serves as the GPU-accelerated rendering backend for Aleph One's geometry and sprite pipeline.

## Core Responsibilities
- Coordinate GPU-accelerated rendering of BSP tree geometry (floors, ceilings, walls, objects)
- Manage OpenGL framebuffer objects and shader rasterizer state setup
- Create and manage TextureManager instances for wall and sprite textures
- Implement clipping, viewport transforms, and endpoint storage for rasterization
- Render viewer-relative sprites (HUD elements, weapons, foreground objects) in rendering tree
- Apply per-surface visual effects (blur, weapon flare, self-luminosity pulsation)
- Override base rasterizer methods to delegate to shader-based implementations

## External Dependencies
- **Base class:** RenderRasterizerClass (RenderRasterize.h) ΓÇö defines core rasterizer interface
- **Geometry types:** sorted_node_data, polygon_data, horizontal_surface_data, vertical_surface_data, endpoint_data, side_data (map.h)
- **Rendering types:** render_object_data, clipping_window_data, rectangle_definition, shape_descriptor (render.h, map.h)
- **GPU backend:** Rasterizer_Shader_Class (Rasterizer_Shader.h) ΓÇö shader pipeline implementation
- **Texture management:** TextureManager (OGL_Textures.h) ΓÇö handles OpenGL texture allocation and state
- **Framebuffer objects:** FBO, FBOSwapper (OGL_FBO.h) ΓÇö GPU render target management
- **Utilities:** Blur class (forward declared; implementation elsewhere), std::unique_ptr (memory)
- **Enums/Constants:** RenderStep (kDiffuse, kGlow pass identifiers), world_distance, long_vector2d, shape_descriptor

# Source_Files/RenderMain/RenderSortPoly.cpp
## File Purpose

Implements depth-based polygon sorting for the rendering pipeline. Transforms an unordered visibility tree into an ordered list of polygons with computed clipping windows. Part of the Aleph One game engine's 3D rendering system.

## Core Responsibilities

- Sort polygons into depth order by recursively traversing the visibility tree
- Build clipping windows (left/right/top/bottom screen bounds) for each sorted polygon
- Calculate vertical clipping data (top and bottom edge constraints) for rendering
- Manage dynamic reallocation of sorted nodes and clipping windows with pointer fixup
- Maintain mapping between polygon indices and sorted node pointers

## External Dependencies

- **Includes:** `cseries.h` (base utilities, macros), `map.h` (map/polygon data), `RenderSortPoly.h` (header)
- **Standard library:** `<string.h>` (memmove), `<limits.h>` (SHRT_MAX, SHRT_MIN)
- **External symbols:** 
  - `get_polygon_data(index)` ΓÇô from map subsystem
  - `RenderVisTreeClass::NodeList`, `node_data` ΓÇô visibility tree structure
  - `clipping_window_data`, `endpoint_clip_data`, `line_clip_data` ΓÇô from render.h
  - `view_data` ΓÇô screen/camera state (from render.h)
  - Macros: `TEST_RENDER_FLAG`, `POINTER_CAST`, `POINTER_DATA`, `vassert`, `csprintf`, `NONE`, `MAX`, `MIN`

# Source_Files/RenderMain/RenderSortPoly.h
## File Purpose
Defines a class for sorting polygons into depth-order for rendering. Converts a visibility tree into sorted nodes, organizing polygons and their clipping windows for efficient rendering from the camera's viewpoint.

## Core Responsibilities
- Sort map polygons into depth order using visibility tree data
- Build clipping windows for viewport culling and scissoring
- Calculate vertical clip bounds for line segments
- Maintain mapping from polygon indices to sorted render nodes
- Accumulate endpoint and line clip data during sorting
- Manage resizable collections of sorted nodes and clipping data

## External Dependencies
- `#include <vector>` ΓÇö STL vector for dynamic arrays
- `#include "world.h"` ΓÇö world coordinate types and macros
- `#include "render.h"` ΓÇö `view_data` struct and render flags
- `#include "RenderVisTree.h"` ΓÇö `node_data`, `clipping_window_data`, `endpoint_clip_data`, `line_clip_data`, `RenderVisTreeClass`
- Symbols defined elsewhere: `render_object_data` (referenced but not defined in bundled headers)

# Source_Files/RenderMain/RenderVisTree.cpp
## File Purpose

Implements the rendering visibility tree class for determining which polygons are visible from the player's viewpoint. Core functionality includes ray casting through the 2D map to build a hierarchical visibility tree, managing polygon visibility, and calculating screen-space clipping information for the renderer.

## Core Responsibilities

- **Build visibility tree**: Construct a tree of visible polygons starting from the viewpoint polygon
- **Ray casting**: Cast rays from view edges and polygon endpoints to find visible adjacent polygons
- **Polygon traversal**: Walk along rays crossing polygon boundaries to determine visibility
- **Clipping calculation**: Compute line and endpoint clipping data for elevation and transparency
- **Endpoint transformation**: Transform world coordinates to screen space for visibility tests
- **Queue management**: Maintain and process a queue of polygons needing visibility checks
- **Automap tracking**: Record visible polygons and lines for automap display

## External Dependencies

- **cseries.h**: Utility types and macros
- **map.h**: Polygon/line/endpoint/side/view data structures; macros for flag testing
- **render.h**: `view_data` structure definition
- **STL**: `<vector>`, `<deque>` for growable containers
- **Defined elsewhere**: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()` (all map accessors)

# Source_Files/RenderMain/RenderVisTree.h
## File Purpose
Defines `RenderVisTreeClass`, which constructs and manages a hierarchical visibility tree for determining which polygons and surfaces are visible from a camera viewpoint. This is a core data structure for the engine's rendering pipeline, originally derived from Marathon's render.c.

## Core Responsibilities
- Build and maintain the render visibility tree through recursive polygon traversal
- Manage viewport clipping boundaries (left, right, top, bottom edges)
- Track screen-space visibility flags for endpoints, lines, and polygons
- Calculate and store clipping information for geometry crossing screen edges
- Provide dynamic resizing of internal data structures (endpoints, lines, clipping data)
- Support polygon sorting trees for depth ordering

## External Dependencies
- **Standard library:** `<deque>`, `<vector>` (STL containers)
- **Engine headers:**
  - `map.h` ΓÇö world geometry (polygon, line, endpoint data structures; `view_data` defined elsewhere)
  - `render.h` ΓÇö `view_data` (camera/viewport state)
- **Implicit:** Type definitions from included headers (world coordinates, vectors, fixed-point math)

# Source_Files/RenderMain/scottish_textures.cpp
## File Purpose
Core software texture rasterizer for the Aleph One game engine. Implements perspective-correct texture mapping for polygons and rectangles using fixed-point math, with support for multiple transfer modes (textured, static, tinted, landscaped) and bit depths (8/16/32-bit). Named after a 1994 internal development comment ("this is not your father's texture mapping library").

## Core Responsibilities
- Allocate and manage DDA line tables and precalculation buffers for rasterization
- Rasterize textured polygons using both horizontal and vertical scanline orientations
- Rasterize clipped and scaled textured rectangles
- Precalculate perspective-correct texture coordinates and shading indices per polygon line/column
- Build Digital Differential Analyzer (DDA) edge tables for polygon boundary tracing
- Support multiple transfer modes and alpha blending strategies (fast/nice)
- Handle landscape-specific texture mapping with tiling and optional vertical repeating
- Dispatch rendering to appropriate bit-depth and alpha-mode template implementations

## External Dependencies
- **Notable includes**: `cseries.h` (common types), `low_level_textures.h` (rendering templates and DDA helpers), `render.h` (view_data, polygon_definition), `Rasterizer_SW.h` (class definition), `preferences.h` (graphics_preferences global), `SW_Texture_Extras.h` (SW_Texture class for opacity)
- **Defined elsewhere**: 
  - `cosine_table[]`, `sine_table[]` ΓÇö trigonometric lookup
  - `TEXBITS_DISPATCH`, `TEXBITS_DISPATCH_2` ΓÇö macros that branch on texture size
  - `texture_horizontal_polygon_lines`, `texture_vertical_polygon_lines`, `landscape_horizontal_polygon_lines`, `randomize_vertical_polygon_lines`, `tint_vertical_polygon_lines` ΓÇö template functions
  - `View_GetLandscapeOptions()` ΓÇö returns landscape rendering options
  - `graphics_preferences` global ΓÇö alpha blending mode setting
  - Global `bit_depth` ΓÇö current rendering depth (8/16/32)
  - Global `screen` in Rasterizer_SW_Class ΓÇö destination bitmap
  - Global `view` in Rasterizer_SW_Class ΓÇö view parameters

# Source_Files/RenderMain/scottish_textures.h
## File Purpose
Defines core texture rendering structures and transfer modes for the Aleph One engine's software and OpenGL renderers. Provides intermediate representations (sprites, wall polygons, floor/ceiling) that bridge game logic and rendering subsystems, with support for tinting, shading, lighting, and 3D model rendering.

## Core Responsibilities
- Define transfer modes for texture rendering (tinted, solid, textured, static, shadeless, landscaped)
- Provide tinting/shading table structures for 8-bit, 16-bit, and 32-bit color rendering
- Define `rectangle_definition` struct for sprites and floor/ceiling textures with position, depth, clipping, and opacity
- Define `polygon_definition` struct for wall polygons with vertices, origin, and shading
- Support OpenGL 3D model rendering via shape descriptors and model pointers
- Manage ambient and ceiling lighting for both software and OpenGL paths
- Track world-space positioning and projected depth for proper lighting and sorting

## External Dependencies
- **cseries.h**: Basic types (`pixel8`, `pixel16`, `pixel32`, `_fixed`), platform macros
- **OGL_Headers.h**: OpenGL/GLEW headers
- **world.h**: World geometry (`world_point3d`, `world_vector3d`, `long_point3d`)
- **shape_descriptors.h**: Shape descriptor type and collection/shape/clut macros
- **OGL_ModelData**: Opaque class (forward-declared; defined elsewhere) for 3D model state
- **Defined elsewhere**: `allocate_texture_tables()` implementation in `SCOTTISH_TEXTURES.C`

# Source_Files/RenderMain/shape_definitions.h
## File Purpose
Defines the in-memory layout and global storage for game shape collections (animated sprites, textures, scenery, enemies, etc.). Serves as the bridge between disk-resident collection files and engine rendering systems, using a 32-byte on-disk header format.

## Core Responsibilities
- Define `collection_header` structure for collection metadata and resource pointers
- Declare the global array of all loaded collection headers indexed by collection ID
- Establish layout contract for variable-format shape/shading data stored on disk or in memory
- Provide constants and structure sizes for disk I/O operations

## External Dependencies
- **Includes:**
  - `shape_descriptors.h` ΓÇô collection IDs, shape descriptor bit-packing macros, and `MAXIMUM_COLLECTIONS` constant
- **Forward declares:**
  - `collection_definition` ΓÇô actual shape geometry/metadata (defined elsewhere)
- **Types used:**
  - `int16`, `uint16`, `int32`, `byte`, `std::vector<byte>` (from cstypes.h, indirectly)


# Source_Files/RenderMain/shape_descriptors.h
## File Purpose
Defines the 16-bit packed `shape_descriptor` type and related macros for encoding/decoding graphical asset references in the Aleph One game engine. Enumerates all sprite collections (enemies, items, scenery, environment) used throughout the game.

## Core Responsibilities
- Define the `shape_descriptor` typedef as a packed 16-bit value (8-bit shape + 5-bit collection + 3-bit CLUT)
- Provide extraction and construction macros for descriptor components
- Enumerate 32 game asset collections (enemies, weapons, items, scenery, landscapes, etc.)
- Define bit widths and maximum values for each descriptor field
- Support color lookup table (CLUT) encoding alongside shape/collection data

## External Dependencies
- `#include "cstypes.h"` ΓÇô provides `uint16` type

# Source_Files/RenderMain/shapes.cpp
## File Purpose
Manages loading and runtime access to game sprite collections (shapes), including bitmap data, color tables, shading tables, and special effects like infravision tinting. Converts compressed shape data from disk into renderable SDL surfaces and maintains lookup tables for light levels and visual effects.

## Core Responsibilities
- Load shape collections from disk (Marathon 1 & 2 formats) and parse metadata, bitmaps, color tables, and animation definitions
- Build and maintain shading tables for multiple bit depths (8/16/32-bit), supporting dynamic darkness/lighting
- Handle RLE (run-length encoded) bitmap decompression and coordinate transformations (mirroring)
- Convert shape data to SDL surfaces with proper color palettes and transparency
- Manage infravision tinting system for light-enhancement goggles effect
- Support runtime shape patching (replacing/modifying shapes after initial load)
- Provide accessor API for other systems to retrieve shape data by collection and index

## External Dependencies
- **SDL2** ΓÇô `SDL_RWops`, `SDL_ReadBE*()`, `SDL_MapRGB()`, `SDL_Surface`, `SDL_CreateRGBSurfaceFrom()`, pixel format queries
- **Game types** ΓÇô `collection_definition`, `bitmap_definition`, `low_level_shape_definition`, `rgb_color_value` (from collection_definition.h)
- **File I/O** ΓÇô `OpenedFile`, `OpenedResourceFile`, `LoadedResource`, `FileHandler.h`
- **Rendering** ΓÇô `OGL_Render.h`, `OGL_LoadScreen.h` for OpenGL integration
- **Configuration** ΓÇô `InfoTree.h` for XML-based infravision tint parsing
- **Utilities** ΓÇô `Packing.h`, `byte_swapping.h`, `cstypes.h`, `cseries.h`

# Source_Files/RenderMain/SW_Texture_Extras.cpp
## File Purpose
Implements software renderer texture management, specifically handling opacity table construction and configuration parsing for textures. Manages collections of textures indexed by collection and shape descriptors, with support for loading/unloading and MML-based configuration.

## Core Responsibilities
- Build opacity tables for textures from shading table data (supporting 16- and 32-bit color modes)
- Manage texture collections storage and lookup via shape descriptors
- Support three opacity calculation modes (full opacity, average RGB, or maximum channel)
- Parse texture configuration from MML (Marathon Markup Language) XML
- Load and unload opacity tables for entire collections on demand
- Singleton pattern for centralized texture extras management

## External Dependencies
- **`interface.h`:** `get_shape_bitmap_and_shading_table()` macro ΓÇö retrieves bitmap and shading data by shape descriptor
- **`screen.h`:** `MainScreenSurface()` ΓÇö returns SDL surface with pixel format info
- **`scottish_textures.h`:** `MAXIMUM_SHADING_TABLE_INDEXES`, `bitmap_definition`, color/pixel constants
- **`collection_definition.h`:** Collection structure and constants (`NUMBER_OF_COLLECTIONS`)
- **`InfoTree.h`:** XML parsing (`InfoTree`, `children_named()`, `read_indexed()`, `read_attr()`)
- **SDL2:** `SDL_PixelFormat` for color format masks and bit shifts
- **Global:** `bit_depth` (extern), `number_of_shading_tables` (extern, used in `build_opac_table()`)

# Source_Files/RenderMain/SW_Texture_Extras.h
## File Purpose
Manages software-rendered textures with opacity and color lookup information. Provides a singleton interface for loading, unloading, and accessing textures by shape descriptor, and supports MML-based texture configuration parsing.

## Core Responsibilities
- Wrap shape descriptors with opacity type, scale, shift, and opacity lookup tables
- Maintain a per-collection registry of software textures via singleton pattern
- Load and unload texture collections, supporting dynamic resource management
- Build opacity tables for texture rendering calculations
- Parse and reset MML (Marathon Markup Language) texture configurations

## External Dependencies
- **cseries.h** ΓÇô general engine utility macros and type definitions
- **cstypes.h** ΓÇô fixed-width integer types (`uint8`, `int`, `float`)
- **shape_descriptors.h** ΓÇô `shape_descriptor` typedef, `NUMBER_OF_COLLECTIONS` macro, collection enumeration
- **\<vector\>** ΓÇô C++ standard `std::vector` container
- **InfoTree** ΓÇô forward-declared; defined elsewhere (likely a markup/config tree class)

# Source_Files/RenderMain/textures.cpp
## File Purpose
Low-level bitmap memory management and pixel remapping for the Aleph One game engine. Handles initialization of bitmap row address tables, pixel data origin calculation, and color table remapping for both linear and RLE-compressed bitmap formats.

## Core Responsibilities
- Calculate bitmap pixel data origin based on structure layout and row addressing mode
- Precalculate row address lookup tables for quick per-row access
- Support both row-major and column-major bitmap layouts
- Handle RLE (Run-Length Encoded) compressed bitmap formats (MARATHON1 and MARATHON2 variants)
- Apply color remapping tables to bitmap pixels using lookup-table substitution
- Manage endianness for big-endian RLE metadata

## External Dependencies
- **Includes:** `cseries.h` (base types, endianness detection), `textures.h` (bitmap_definition).
- **External symbols:** `pixel8`, `byte`, `int32`, `int16`, `uint16` (defined in cseries/cstypes.h); `NONE` (likely a -1 constant).
- **Preprocessor:** `MARATHON1`/`MARATHON2` conditionals select RLE format variant.

# Source_Files/RenderMain/textures.h
## File Purpose
Defines bitmap and texture metadata structures and utility functions for the rendering pipeline. Provides bitmap definition buffers, pixel data mapping, and row-address calculation for texture processing in the Aleph One game engine.

## Core Responsibilities
- Define bitmap metadata structure (`bitmap_definition`) with pixel dimensions, row stride, and flags
- Provide RAII bitmap buffer class for safe allocation of bitmap definition + row pointer arrays
- Declare bitmap origin and row-address calculation functions
- Declare pixel data remapping functions for color/palette transformations
- Define bitmap flags for rendering hints (column order, transparency, patching)

## External Dependencies
- `cseries.h` ΓÇö core platform abstractions, type definitions (pixel8, byte, int16, uint16)
- `<vector>` ΓÇö standard library for dynamic byte buffer
- Defined elsewhere: `pixel8` (typedef for uint8), bitmap processing implementations in TEXTURES.C

# Source_Files/RenderMain/vec3.h
## File Purpose
Defines lightweight vector and matrix types (vec3, vec4, vertex2, vertex3, mat4) for 3D graphics computations in the Aleph One engine. Provides basic linear algebra operations (dot product, cross product, normalization) and OpenGL integration for matrix transformations.

## Core Responsibilities
- Define homogeneous vector and vertex types for 3D graphics
- Implement vector arithmetic operators (+, ΓêÆ, ├ù, scalar multiplication)
- Provide vector math utilities (dot product, cross product, normalization)
- Define 4├ù4 matrix type with OpenGL interop
- Establish floating-point comparison tolerance for vector equality
- Support both direction vectors (w=0) and position vertices (w=1)

## External Dependencies
- **OGL_Headers.h** ΓÇô Provides OpenGL function declarations and type definitions (GLfloat, GLenum, GL_* constants)
- **cfloat** ΓÇô Provides FLT_MIN for threshold calculation
- **cmath** ΓÇô Provides std::sqrt, std::abs for vector math


