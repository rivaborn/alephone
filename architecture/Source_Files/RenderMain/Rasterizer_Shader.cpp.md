# Source_Files/RenderMain/Rasterizer_Shader.cpp

## File Purpose
Implements shader-based OpenGL rasterization for Aleph One, managing camera transformations, projection matrices, framebuffer composition, and post-processing effects like gamma correction.

## Core Responsibilities
- Configure projection and modelview matrices from view parameters (FOV, position, orientation)
- Compute landscape shader transformation for proper texture alignment
- Manage framebuffer swapping for offscreen rendering and composition
- Apply gamma correction and void-smearing effects during frame output
- Handle view distortion during teleport effects

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Rasterizer_Shader_Class` | class | Inherits from `Rasterizer_OGL_Class`; overrides view/render pipeline methods |
| `FBOSwapper` | class (opaque) | Manages offscreen framebuffer swapping and composition |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FixedAngleToDegrees` | const float | static | Convert engine angle units to degrees (360 / (FIXED_ONE * FULL_CIRCLE)) |
| `kViewBaseMatrix` | const GLdouble[16] | static | Base rotation matrix aligning view space to world space |
| `kViewBaseMatrixInverse` | const GLdouble[16] | static | Inverse of base matrix; used for landscape shader setup |
| `MAXIMUM_VERTICES_PER_WORLD_POLYGON` | macro | static | Polygon vertex limit with 4-unit margin |

## Key Methods

### Rasterizer_Shader_Class::SetView
- **Signature:** `void SetView(view_data& view)`
- **Purpose:** Configure projection and modelview matrices; compute and upload landscape shader matrices; handle view distortion during teleports.
- **Inputs:** 
  - `view` (view_data&): Screen dimensions, FOV, camera origin, yaw/pitch angles, world-to-screen scaling, perspective mode flag
- **Outputs/Return:** None; modifies OpenGL state (matrices, uniform variables in active shaders)
- **Side effects:** 
  - Recreates `swapper` (FBOSwapper) if screen dimensions changed
  - Calls `glMatrixMode`, `glLoadIdentity`, `glFrustum`, `glRotated`, `glTranslated`, `glMultMatrixd`, `glGetFloatv`
  - Enables and updates landscape shaders (`S_Landscape`, `S_LandscapeBloom`) with computed matrix uniforms
- **Calls:** `OGL_SetView()`, `View_FOV_FixHorizontalNotVertical()`, `Shader::get()`, `s->enable()`, `s->setMatrix4()`, `Shader::disable()`, `MainScreenPixelScale()`
- **Notes:** 
  - FOV can be horizontal or vertical (controlled by preference)
  - Handles pitch clamping to [-180, 180]
  - Applies view.mimic_sw_perspective flag for software renderer compatibility
  - Computes ytan/xtan for frustum from tangent of half-FOV; adjusts for aspect ratio
  - Landscape matrix computed separately to align textures during standard pitch ranges

### Rasterizer_Shader_Class::setupGL
- **Signature:** `void setupGL()`
- **Purpose:** Initialize OpenGL state for shader-based rasterization; determine if void-smearing is needed.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:**
  - Resets `view_width`, `view_height` to 0
  - Clears `swapper` (reset to null)
  - Sets `smear_the_void` flag based on OGL_Flag_VoidColor configuration
- **Calls:** `Get_OGL_ConfigureData()`, `TEST_FLAG()`
- **Notes:** Called during initialization; enables void-smearing if void-color rendering is disabled.

### Rasterizer_Shader_Class::Begin
- **Signature:** `void Begin()`
- **Purpose:** Prepare for frame rendering; activate offscreen framebuffer and restore prior frame if void-smearing is enabled.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:**
  - Calls parent `Rasterizer_OGL_Class::Begin()`
  - Activates framebuffer via `swapper->activate()`
  - Draws prior framebuffer contents if `smear_the_void` is true
- **Calls:** `Rasterizer_OGL_Class::Begin()`, `swapper->activate()`, `swapper->current_contents().draw_full()`
- **Notes:** Void-smearing preserves background when rendering transparent void regions.

### Rasterizer_Shader_Class::End
- **Signature:** `void End()`
- **Purpose:** Composite final framebuffer to screen with gamma correction applied.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:**
  - Deactivates and swaps framebuffers
  - Applies gamma correction shader if gamma_level deviates from 1.0
  - Renders final composited frame to screen
  - Calls parent `Rasterizer_OGL_Class::End()`
- **Calls:** `swapper->deactivate()`, `swapper->swap()`, `get_actual_gamma_adjust()`, `Shader::get(Shader::S_Gamma)`, `s->enable()`, `s->setFloat()`, `swapper->draw()`, `Shader::disable()`, `SetForeground()`, `glColor3f()`, `OGL_RenderFrame()`, `Rasterizer_OGL_Class::End()`
- **Notes:** Gamma correction only applied if adjustment differs from 1.0┬▒0.01; renders full black quad under framebuffer as safety clear.

## Control Flow Notes

- **Initialization:** `setupGL()` called once at engine startup.
- **Per-frame:** `SetView()` called before rendering to set up matrices; `Begin()` activates offscreen buffer; game world rendered; `End()` composites to screen with post-processing.
- **Gating:** Entire file wrapped in `#ifdef HAVE_OPENGL`, so shader rasterizer only compiles when OpenGL support is enabled.

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
