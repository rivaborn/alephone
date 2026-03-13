# Source_Files/RenderMain/RenderRasterize_Shader.cpp - Enhanced Analysis

## Architectural Role

This file implements the shader-based **rendering backend** for Aleph One, one of three rasterizer strategies (alongside software and classic OpenGL). It acts as the bridge between the **RenderVisTree/RenderSortPoly** visibility/sorting layer and the GPU, translating sorted polygons and objects into shader-driven draw calls. Critically, it orchestrates **multi-pass rendering** (diffuse + glow) and **postprocessing** (bloom/blur), making it responsible for the engine's distinctive visual effects pipelineΓÇöeffects that are impossible in the simpler software/classic backends.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderMain::render()** (render.cpp) ΓÇô selects and invokes `render_tree()` as the active rasterizer backend per frame
- **RenderRasterizerClass** (base class, RenderRasterize.h) ΓÇô dispatches calls to `render_node()`, `clip_to_window()`, `render_node_object()` via virtual overrides
- **Rasterizer_Shader_Class** (Rasterizer_Shader.h) ΓÇô parent class holding FBOSwapper state; setupGL receives instance reference

### Outgoing (what this file depends on)
- **Shader system** (OGL_Shader.h) ΓÇô `Shader::get()`, `Shader::loadAll()`, `enable()`, `setFloat()` for all uniform binding; ~20 shader programs retrieved and configured
- **Texture/rendering infrastructure** (OGL_Textures.h, TextureManager) ΓÇô `Setup()`, `RenderNormal()`, `RenderBump()`, `RenderGlowing()` for polygon/sprite rendering
- **Game state globals** ΓÇô `current_player` (infravision/weapons), `view` (camera/ticks), `cosine_table`/`sine_table` (animation lookups)
- **World geometry callbacks** ΓÇô base class invokes `render_node_side()`, `render_node_floor_or_ceiling()` with polygon data

## Design Patterns & Rationale

**1. Strategy Pattern (Rasterizer hierarchy)**  
Three backends (`RenderRasterize_SW`, `RenderRasterize_OGL`, `RenderRasterize_Shader`) implement the same interface, selected at startup based on GPU capabilities. This file provides the most feature-rich implementation (bloom, multi-pass, shader effects).

**2. Template Method (render_tree override)**  
Wraps parent `RenderRasterizerClass::render_tree(kDiffuse)` with shader setup before and bloom postprocessing after. Separates concerns: parent handles polygon/object iteration, this file handles GPU-specific rendering strategy.

**3. Builder/Configuration Pattern (TextureManager setup)**  
`setupWallTexture()` and `setupSpriteTexture()` encapsulate a complex decision tree (transfer modes, infravision, landscape sphere mapping) into a single `TextureManager` instance. Keeps shader selection logic localized and maintainable.

**4. Framebuffer Postprocessing Pipeline**  
`Blur` class manages off-screen FBO with multi-pass filtering (horizontal blur ΓåÆ vertical blur ΓåÆ bloom composite). Ping-pong FBOs reduce memory bandwidth vs. reading back to CPU.

**Why this structure?**  
- **Separation of concerns**: Base class owns BSP traversal and per-polygon callbacks; this file owns GPU state and effect pipeline  
- **Shader decoupling**: Complex transfer mode logic ΓåÆ shader selection; actual blending/animation in GLSL  
- **Conditional postprocessing**: Bloom disabled during infravision (incompatible visual modes); saves GPU time on low-end systems

## Data Flow Through This File

**Per-frame entry point: `render_tree()`**
```
GameWorld state (current_player, view, polygon/object data)
    Γåô
Calculate frame-global uniforms:
  - Camera yaw/pitch ΓåÆ radians (via FixedAngleToRadians constant)
  - Weapon flare, self-luminosity (from player powerup intensity)
  - Fog settings (from OGL_GetCurrFogData global)
  - Clipping window bounds (min/max X from all windows)
    Γåô
Enable 6 landscape shaders + 12 fog-mode shaders
Set uniform: U_FogMix, U_Yaw, U_Pitch, U_FogMode (uniform batching)
    Γåô
RenderRasterizerClass::render_tree(kDiffuse)
  Γö£ΓöÇ Per polygon/side ΓåÆ setupWallTexture(shape_desc, transfer_mode, pulsate, wobble, intensity)
  Γöé   Γö£ΓöÇ Shader selection: S_Landscape/S_LandscapeSphere/S_Bump/S_Wall based on transfer mode + infravision
  Γöé   Γö£ΓöÇ TextureManager setup + texture binding
  Γöé   ΓööΓöÇ glDrawArrays with diffuse shader
  Γöé
  ΓööΓöÇ Per sprite ΓåÆ setupSpriteTexture(rect, type, offset)
      Γö£ΓöÇ Shader selection: S_Sprite/S_Invincible/S_Invisible based on transfer mode
      Γö£ΓöÇ Uniform assignment: U_Flare, U_SelfLuminosity, U_Depth
      ΓööΓöÇ glDrawArrays with diffuse shader

render_viewer_sprite_layer(kDiffuse)
  ΓööΓöÇ Per HUD weapon ΓåÆ render_viewer_sprite() ΓåÆ setupSpriteTexture() ΓåÆ glDrawArrays
    Γåô
[If bloom enabled AND no infravision]:
    blur->begin() ΓåÉ activate FBO, disable sRGB blending
      Γåô
      RenderRasterizerClass::render_tree(kGlow)  ΓåÉ re-render entire scene with glow shaders
      render_viewer_sprite_layer(kGlow)
      Γåô
    blur->end() ΓåÉ swap FBO
    blur->draw(*swapper) ΓåÉ 5-pass Gaussian blur (HΓåÆVΓåÆcomposite) with additive blending
    Γåô
Screen display ΓåÉ composite bloom-blurred layer over main framebuffer
```

**Key state transformations:**
- Transfer modes (enum) ΓåÆ shader program (string lookup via `Shader::get()`)
- Animation phase (viewΓåÆtick_count, fixed-angle offset) ΓåÆ texture matrix transforms (via AnimTxtr_Translate)
- Intensity values (0..FIXED_ONE) ΓåÆ normalized floats (0..1) for shader uniforms

## Learning Notes

**What studying this file teaches:**
1. **Modern multi-pass GPU rendering** ΓÇô how post-2010s engines use off-screen FBOs and ping-pong buffering for effects like bloom/blur, not compositing on CPU
2. **Shader state management** ΓÇô uniform batching (set many shaders upfront), shader swapping overhead, per-draw state changes
3. **Legacy material system** ΓÇô Marathon's rich "transfer mode" enum (static, tinted, invisible, landscape, etc.) predates PBR; shows how older engines encoded material intent
4. **Fixed-point arithmetic in real-time rendering** ΓÇô game stores angles/positions as fixed-point for deterministic networking; shaders convert to float
5. **Hierarchical rendering callbacks** ΓÇô base class (visibility/sorting) knows nothing about shader details; derived class intercepts callbacks to add GPU-specific logic

**Idiomatic patterns for Aleph One (ca. 2009):**
- **Per-object shader selection during traversal** vs. modern batching by material-first
- **Dynamic shader/uniform assignment** ΓÇô no static pipelines; decision trees live in C++ per-polygon
- **Frame-global uniform setup** ΓÇô uniforms set once on many shaders to avoid repetition
- **Conditional effect toggling** ΓÇô bloom, bump mapping, infravision all toggled as boolean flags in a single code path (not separate rendering pipelines)

## Potential Issues

1. **State machine fragility**: `setupWallTexture()` and `setupSpriteTexture()` enable shaders but don't save/restore previous shader state. If called recursively or from nested contexts, the last `enable()` wins. Works because they're called top-down, but fragile.

2. **Hardcoded bloom resolution**: `Blur(640., ...)` is independent of screen resolution. On 4K displays, 640px blur texture is severely undersampled. Modern engines scale bloom FBO to screen dimensions.

3. **Infravision coupling**: Bloom postprocessing is entirely disabled if `current_player->infravision_duration > 0`. This is a one-way dependency (rendering depends on game state) that could be cleaner as a config flag passed to `render_tree()`.

4. **Per-polygon shader swapping cost**: For each polygon, `setupWallTexture()` calls `Shader::get()`, `s->enable()`, and 3+ `setFloat()` calls. Modern engines would sort by shader first, batch draw calls. For a 1000-polygon level, this incurs 1000 draw call submissions and uniform updates.

5. **Matrix stack (deprecated GL)**: `clip_to_window()` uses `glPushMatrix()`/`glPopMatrix()`, legacy fixed-function remnants. Modern shaders use explicit MVP matrices, but this works fine for screen-space clipping.
