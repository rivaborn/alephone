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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Rasterizer_Shader_Class | class | Main shader-based rendering interface; extends Rasterizer_OGL_Class |
| FBOSwapper | class (forward decl.) | Manages framebuffer object swapping; implementation in separate file |

## Global / File-Static State
None.

## Key Functions / Methods

### Rasterizer_Shader_Class (constructor)
- Signature: `Rasterizer_Shader_Class()`
- Purpose: Initialize the shader rasterizer, including FBO swapper and rendering state
- Inputs: None
- Outputs/Return: Initialized instance
- Side effects: Allocates FBOSwapper instance; likely initializes GL state
- Calls: FBOSwapper constructor (inferred)
- Notes: Defined in implementation file; called once per rendering session

### ~Rasterizer_Shader_Class (destructor)
- Signature: `~Rasterizer_Shader_Class()`
- Purpose: Clean up shader rasterizer resources, especially FBO swapper
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates FBOSwapper via unique_ptr; may clean up GL resources
- Calls: FBOSwapper destructor (via unique_ptr)
- Notes: Virtual (implied by base class pattern); ensures proper resource cleanup

### SetView
- Signature: `virtual void SetView(view_data& View)`
- Purpose: Configure the rendering viewport and projection for a given camera view
- Inputs: Reference to view_data structure (camera position, orientation, FOV)
- Outputs/Return: None
- Side effects: Updates view_width, view_height; configures GL projection/modelview matrices
- Calls: Overrides base class; likely calls OGL_SetView or equivalent
- Notes: Must be called before rendering; updates cached viewport dimensions

### setupGL
- Signature: `virtual void setupGL()`
- Purpose: Initialize or reset OpenGL state specific to shader-based rendering
- Inputs: None
- Outputs/Return: None
- Side effects: Configures GL state (shaders, uniforms, blending, depth testing, etc.)
- Calls: GL state configuration functions (not visible in header)
- Notes: Called during initialization or when GL context changes

### Begin
- Signature: `virtual void Begin()`
- Purpose: Start a rendering frame; prepare FBO and state for drawing
- Inputs: None
- Outputs/Return: None
- Side effects: Activates FBO swapper; clears buffers; sets GL state for frame
- Calls: FBOSwapper methods; GL state setup functions
- Notes: Paired with End(); must be called before any geometry rendering

### End
- Signature: `virtual void End()`
- Purpose: Finalize a rendering frame; swap framebuffers and resolve rendering
- Inputs: None
- Outputs/Return: None
- Side effects: Swaps or resolves FBO; may submit GL commands to GPU
- Calls: FBOSwapper methods; GL presentation functions
- Notes: Paired with Begin(); called after all geometry is rendered

## Control Flow Notes
This class fits into the **render phase** of the game loop:
1. **Initialization**: Constructor allocates FBOSwapper; setupGL configures GL state
2. **Per-Frame**: SetView configures camera ΓåÆ Begin starts frame ΓåÆ rendering (via inherited methods) ΓåÆ End finalizes frame
3. **Shutdown**: Destructor cleans FBOSwapper and GL resources

The Begin/End pattern brackets all rendering operations for a single frame, allowing FBO swapping and state management around geometry submission.

## External Dependencies
- **Rasterizer_OGL.h**: Base class `Rasterizer_OGL_Class` providing core OpenGL rendering interface
- **cseries.h**: Cross-platform utilities (types, macros)
- **map.h**: World/map data structures (used by view_data)
- **std::memory**: `std::unique_ptr<FBOSwapper>` for RAII resource management
- **FBOSwapper** (forward declared): Frame buffer object swapper; definition elsewhere
- **view_data** (from map.h or related): Camera view configuration structure
- **RenderRasterize_Shader** (friend class): Privileged access to shader rasterizer internals

Compilation guarded by `#ifdef HAVE_OPENGL` ΓÇö not available if OpenGL support is disabled.
