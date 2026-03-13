# Source_Files/RenderMain/Rasterizer_Shader.cpp - Enhanced Analysis

## Architectural Role

This file implements one of three rasterizer backends for Aleph One, specializing in GPU-accelerated shader-based rendering. It sits at a critical junction between the rendering pipeline and OpenGL: it prepares the GPU state (projection/view matrices, framebuffer bindings) and orchestrates post-processing (gamma correction, void preservation) for each frame. Unlike the software rasterizer, it's output-agnostic about polygon rasterization itselfΓÇöthat's delegated to fragment shaders and happens after SetView/Begin return. This design enables the rendering pipeline to remain backend-agnostic while leveraging GPU capabilities for complex effects.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderMain/render.cpp** - Calls SetView/Begin/End in main frame loop via polymorphic `Rasterizer*` interface
- **RenderRasterize_Shader.cpp** - Likely consumes the matrices set by SetView to drive GPU polygon rasterization
- **Shader system** (OGL_Shader.h/cpp) - Landscape shaders (S_Landscape, S_LandscapeBloom) receive inverse matrix uniforms computed here; gamma shader receives adjustment factor

### Outgoing (what this file depends on)
- **OGL_FBO.h/cpp** - FBOSwapper manages offscreen framebuffer ping-pong; activates/deactivates/swaps at frame boundaries
- **OGL_Shader.h/cpp** - Enables shaders and updates their uniform variables (setMatrix4, setFloat); assumes Shader singleton registry exists
- **Preferences/screen.h** - Reads gamma_level from graphics_preferences; calls OGL_RenderFrame for final screen blit and SetForeground for state reset
- **World state** (player.h, ChaseCam.h) - Accesses view.origin (camera position) and view virtual_yaw/pitch (camera orientation) from view_data struct
- **OGL_Headers.h** - Platform OpenGL context (glMatrixMode, glFrustum, glRotated, glGetFloatv, etc.)

## Design Patterns & Rationale

**Polymorphic Rasterizer Backend**  
Inherits from `Rasterizer_OGL_Class`, overriding SetView/Begin/End. This Template Method pattern decouples the rendering pipeline from backend details while allowing specialization (SetView is different from software rasterizerΓÇöcomputes matrices for GPU instead of per-pixel dispatch).

**Framebuffer Ping-Pong (FBOSwapper)**  
The pattern: activate offscreen FBO ΓåÆ render world ΓåÆ deactivate ΓåÆ swap buffers ΓåÆ composite to screen. This enables:
- Post-processing (gamma correction applied to entire frame)
- Void-smearing (background preservation via `draw_full()`)
- Temporal coherence (previous frame available for blending/smear)

Rationale: the software rasterizer doesn't need this level of buffering; GPU rasterization naturally produces full frames suitable for composition.

**Lazy Matrix Updates**  
SetView recomputes matrices *per-frame* rather than caching them. This is acceptable because:
- Matrix math is cheap (~dozen floating-point ops)
- View changes every frame in mouselook scenarios
- Shader uniforms must be updated anyway, so amortizing setup cost is minimal

**Separate Landscape Shader Matrix**  
The code computes `landscapeInverseMatrix` separately and uploads to both `S_Landscape` and `S_LandscapeBloom`. This suggests:
- Landscape (sky/dome) textures need inverse transformation to align properly
- Standard perspective projection would stretch/distort these textures during pitch changes
- A dedicated matrix avoids expensive texture coordinate adjustments in the fragment shader

Rationale: landscapes are typically pre-rendered at standard pitch angles; the inverse matrix "undoes" the camera pitch to keep them visually stable (a common technique in 3D engines for skyboxes).

**Software Perspective Emulation**  
The `mimic_sw_perspective` flag and associated yoff/xoff adjustments preserve visual compatibility with the original software rasterizer. This is a compatibility requirement, not a correctness one.

## Data Flow Through This File

```
setupGL() [initialization]
  Γö£ΓöÇ Clear FBOSwapper, set smear_the_void flag based on config
  ΓööΓöÇ (happens once at startup)

Per-Frame Loop:
  Γö£ΓöÇ SetView(view_data)
  Γöé  Γö£ΓöÇ Query FOV preference (horizontal vs. vertical fixed)
  Γöé  Γö£ΓöÇ Compute tangent-based frustum (near/far planes hardcoded)
  Γöé  Γö£ΓöÇ Apply view distortion from teleport effects (real_world_to_screen scaling)
  Γöé  Γö£ΓöÇ Compute landscape inverse matrix and upload to S_Landscape, S_LandscapeBloom shaders
  Γöé  ΓööΓöÇ Compute main modelview matrix (camera position + rotation + base matrix)
  Γöé
  Γö£ΓöÇ Begin()
  Γöé  Γö£ΓöÇ Call parent Rasterizer_OGL_Class::Begin() [OGL state setup]
  Γöé  Γö£ΓöÇ Activate FBOSwapper ΓåÆ bind offscreen framebuffer
  Γöé  ΓööΓöÇ If smear_the_void: redraw previous frame (preserves background for transparency)
  Γöé
  Γö£ΓöÇ [Rendering pipeline drives GPU (polygons, sprites, effects)]
  Γöé  ΓööΓöÇ Shaders use matrices set by SetView
  Γöé
  ΓööΓöÇ End()
     Γö£ΓöÇ Deactivate FBOSwapper ΓåÆ unbind offscreen framebuffer
     Γö£ΓöÇ Swap FBOSwapper ΓåÆ flip ping-pong buffers
     Γö£ΓöÇ If gamma_level Γëá 1.0 ┬▒ 0.01: enable S_Gamma shader, upload adjustment, render screen quad
     Γö£ΓöÇ Else: render FBOSwapper contents directly
     Γö£ΓöÇ SetForeground() + glColor3f(0,0,0) + OGL_RenderFrame() [final composite to screen]
     ΓööΓöÇ Call parent Rasterizer_OGL_Class::End() [state cleanup]
```

**Key state transitions:**
- `smear_the_void` flag set once in setupGL; toggled by config at initialization
- `swapper` (FBOSwapper) recreated if screen dimensions change; otherwise reused across frames
- Landscape shader uniforms updated every frame; main matrices updated every frame

## Learning Notes

**Fixed-Point Angle System**  
The constant `FixedAngleToDegrees = 360.0/(FIXED_ONE * FULL_CIRCLE)` reveals that Aleph One (like Quake/Doom) uses fixed-point angle units internally. This era of engines avoided floating-point trigonometry where possible; note the code doesn't call `sin/cos` directly but pre-computes tangents from degrees.

**Frustum Computation**  
Rather than using `gluPerspective`, the code manually computes half-widths/heights (`xtan`, `ytan`) from FOV and builds the frustum directly. This provides fine-grained control for:
- Aspect ratio adjustment per FOV mode
- Teleport-induced distortion (real_world_to_screen scaling)
- Software renderer parity (mimic_sw_perspective adds an offset to avoid y-stretch)

**Idiomatic OpenGL (circa 2009)**  
The use of fixed-function pipeline (glMatrixMode, glLoadIdentity, glMultMatrixd) is direct from OpenGL 1.x. By 2009, this was already deprecated in favor of shaders, but Aleph One maintains backward compatibility. The fact that *landscape* shaders still read matrices as uniforms rather than computing them suggests a hybrid approachΓÇösome geometry uses pure shaders, landscapes may use fixed-function or custom matrix handling.

**Void Smearing as a Visual Technique**  
The void (areas outside map geometry) can be rendered as transparent or a solid color. Smearing preserves the previous frame in transparent voids, creating a visual continuity effect. This is disabled when `OGL_Flag_VoidColor` is set (solid void color preferred). Modern engines would use a skybox or environmental lighting instead.

## Potential Issues

**Missing Bounds on Hardcoded Planes**  
`nearVal = 64.0` and `farVal = 128.0 * 1024.0` are literals with no relation to visible world geometry. If a map has:
- Very distant geometry beyond 131,072 units, it clips
- Very close geometry below 64 units (e.g., weapon model), it clips

No Z-fighting mitigation (like log-space depth) is visible. This is acceptable for a closed game (Marathon's maps are bounded) but fragile for modded content.

**Incomplete Gamma Precision**  
The tolerance check `if (gamma_adj < 0.99f || gamma_adj > 1.01f)` skips the gamma shader for ┬▒1% adjustments. This saves GPU work but means no real-time gamma changes in that bandΓÇöminor issue but worth noting.

**Unused Includes**  
`lightsource.h`, `media.h`, `player.h` (except for view.origin/yaw/pitch), `weapons.h` are included but not directly referenced in this file. These are likely:
- Header chain pollution (included by other headers)
- Legacy dependencies from earlier versions
- Included preemptively for future extensions

**No Validation of Shader State**  
The code calls `Shader::get(Shader::S_Landscape)->setMatrix4(...)` without checking:
- Does the shader exist?
- Is it valid (compiled successfully)?
- Does it have that uniform?

If a shader compilation fails at startup, this will silently fail or crash at runtime. Defensive code would add assertions or error checks.
