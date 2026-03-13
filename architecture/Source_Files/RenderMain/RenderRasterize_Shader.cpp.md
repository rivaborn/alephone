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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Blur` | class | Manages bloom effect via framebuffer swapping and multi-pass Gaussian blur filtering |
| `RenderRasterize_Shader` | class | Main shader-based renderer; inherits from RenderRasterizerClass |
| `ExtendedVertexData` | struct | Extended vertex format with position, texture coords, color, and glow color for HUD rendering |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TWO_PI` | const double | file-static | Mathematical constant (8*atan(1.0)) for angle conversions |
| `FixedAngleToRadians` | const float | file-static | Radians per fixed-angle unit |
| `FixedAngleToDegrees` | const float | file-static | Degrees per fixed-angle unit |
| `Radian2Circle` | const double | file-static | Reciprocal of 2╧Ç for circular math |
| `FullCircleReciprocal` | const double | file-static | Reciprocal of FULL_CIRCLE constant |

## Key Functions / Methods

### Blur::draw
- **Signature**: `void draw(FBOSwapper& dest)`
- **Purpose**: Apply multi-pass Gaussian blur and bloom filtering to captured glow content
- **Inputs**: `dest` ΓÇô destination framebuffer for output
- **Outputs**: Blurred/bloomed framebuffer
- **Side effects**: Executes 5 blur passes (configurable), swaps FBOs, sets blend mode to additive
- **Calls**: `_shader_blur->enable()`, `_shader_bloom->enable()`, `_swapper.filter()`, `dest.blend_multisample()`
- **Notes**: Alternates horizontal/vertical blur directions; bloom shader composites result into destination

### RenderRasterize_Shader::setupGL
- **Signature**: `void setupGL(Rasterizer_Shader_Class& Rasterizer)`
- **Purpose**: One-time initialization of shaders and bloom postprocessing system
- **Inputs**: Parent rasterizer reference
- **Outputs**: None
- **Side effects**: Loads all shaders via `Shader::loadAll()`, creates Blur object if enabled
- **Calls**: `Shader::loadAll()`, `Shader::get()`, `blur.reset()`
- **Notes**: Called once at engine startup after OpenGL and texture systems initialized

### RenderRasterize_Shader::render_tree
- **Signature**: `void render_tree()`
- **Purpose**: Main per-frame render override; executes multi-pass rendering with optional bloom
- **Inputs**: None
- **Outputs**: None
- **Side effects**: Sets shader uniforms, caches clipping window bounds, renders scene in diffuse and glow passes
- **Calls**: `Shader::get()`, `RenderRasterizerClass::render_tree()`, `render_viewer_sprite_layer()`, `blur->begin/end/draw()`
- **Notes**: Calculates `weaponFlare` and `selfLuminosity` from view state; skips bloom if player has infravision

### RenderRasterize_Shader::clip_to_window
- **Signature**: `void clip_to_window(clipping_window_data *win)`
- **Purpose**: Configure OpenGL clipping planes to constrain rendering to a viewport window
- **Inputs**: Clipping window with left/right boundary vectors
- **Outputs**: None
- **Side effects**: Pushes/pops matrix; enables GL_CLIP_PLANE0/1; sets plane equations
- **Calls**: `glPushMatrix()`, `glRotatef()`, `glClipPlane()`, `glEnable/Disable(GL_CLIP_PLANE0/1)`
- **Notes**: Rotates to player yaw orientation; applies ┬▒0.1┬░ fudge factors to avoid edge artifacts

### RenderRasterize_Shader::setupSpriteTexture
- **Signature**: `std::unique_ptr<TextureManager> setupSpriteTexture(const rectangle_definition& rect, short type, float offset, RenderStep renderStep)`
- **Purpose**: Configure texture manager and select appropriate shader for sprite rendering based on transfer mode
- **Inputs**: Rectangle definition, texture type (Inhabitant/WeaponsInHand), depth offset, render step
- **Outputs**: Configured TextureManager unique_ptr
- **Side effects**: Enables GL_TEXTURE_2D; selects shader based on transfer mode; sets shader uniforms for flare, luminosity, depth
- **Calls**: `Shader::get()`, `TMgr->Setup()`, `TMgr->RenderNormal()`, multiple `s->setFloat()` calls
- **Notes**: Handles static (invincible), infravision, tinted (invisible), and textured transfer modes

### RenderRasterize_Shader::setupWallTexture
- **Signature**: `std::unique_ptr<TextureManager> setupWallTexture(const shape_descriptor& Texture, short transferMode, float pulsate, float wobble, float intensity, float offset, RenderStep renderStep)`
- **Purpose**: Configure texture manager and shader for wall/landscape polygon rendering
- **Inputs**: Texture descriptor, transfer mode, animation params, intensity, depth offset, render step
- **Outputs**: Configured TextureManager unique_ptr
- **Side effects**: Complex shader selection; sets texture matrix transforms for landscape UV scaling and sphere mapping
- **Calls**: `get_shape_bitmap_and_shading_table()`, `View_GetLandscapeOptions()`, `Shader::get()`, `s->setFloat()` for uniforms
- **Notes**: Handles landscape sphere mapping; applies azimuth/vertical scaling transforms; sets bloom parameters for glow pass

### instantiate_transfer_mode
- **Signature**: `void instantiate_transfer_mode(view_data *view, short transfer_mode, world_distance &x0, world_distance &y0)`
- **Purpose**: Calculate texture coordinate offset for animated transfer modes (scrolling/wandering)
- **Inputs**: View state, transfer mode enum, output references for x/y offsets
- **Outputs**: Modifies `x0`, `y0` with animated offsets
- **Side effects**: None (read-only computation)
- **Calls**: Trigonometric lookup tables (`cosine_table`, `sine_table`)
- **Notes**: Handles 8 slide variants (fast/slow, forward/reverse, horizontal/vertical) and wander mode; wobble handled in shader

### setupGlow
- **Signature**: `bool setupGlow(view_data *view, TextureManager& TMgr, float wobble, float intensity, float flare, float selfLuminosity, float offset, RenderStep renderStep)`
- **Purpose**: Prepare and render glow layer if texture has glow map
- **Inputs**: Animation and lighting parameters
- **Outputs**: Returns true if glow was rendered
- **Side effects**: Enables GL_TEXTURE_2D, GL_BLEND, GL_ALPHA_TEST; renders glow texture with additive blending
- **Calls**: `TMgr->IsGlowMapped()`, `TMgr->RenderGlowing()`, `setupBlendFunc()`, `Shader::get()`, `s->setFloat()` for bloom parameters
- **Notes**: Only executes if texture manager reports glow map present; selects shader based on texture type

### render_node_floor_or_ceiling
- **Signature**: `void render_node_floor_or_ceiling(clipping_window_data *window, polygon_data *polygon, horizontal_surface_data *surface, bool void_present, bool ceil, RenderStep renderStep)`
- **Purpose**: Render floor or ceiling polygon with animated texture and lighting
- **Inputs**: Clipping window, polygon, surface, void presence flag, ceiling orientation, render step
- **Outputs**: None
- **Side effects**: Builds vertex/texcoord arrays; calls glDrawArrays twice (normal + glow); manages blend modes
- **Calls**: `setupWallTexture()`, `clip_to_window()`, `instantiate_transfer_mode()`, `glVertexPointer()`, `glDrawArrays()`, `setupGlow()`
- **Notes**: Handles ceiling vertex winding order reversal; disables blend if void present

### render_node_side
- **Signature**: `void render_node_side(clipping_window_data *window, vertical_surface_data *surface, bool void_present, RenderStep renderStep)`
- **Purpose**: Render vertical wall polygon
- **Inputs**: Clipping window, vertical surface data, void flag, render step
- **Outputs**: None
- **Side effects**: Builds 3D quad vertices with proper texcoord mapping; calls glDrawArrays twice
- **Calls**: `setupWallTexture()`, `instantiate_transfer_mode()`, `clip_to_window()`, `glVertexPointer()`, `glDrawArrays()`, `setupGlow()`
- **Notes**: Computes texture coordinate offsets for 2x/4x tiling modes; calculates tangent/normal for per-vertex transforms

### RenderModel
- **Signature**: `bool RenderModel(rectangle_definition& RenderRectangle, short Collection, short CLUT, float flare, float selfLuminosity, RenderStep renderStep)`
- **Purpose**: Render 3D model with optional animation, bump mapping, and glow
- **Inputs**: Rectangle definition, collection/CLUT indices, flare/luminosity parameters, render step
- **Outputs**: Returns false if skin unavailable
- **Side effects**: Extensive; sets culling, loads textures, calls glDrawElements multiple times for normal and glow
- **Calls**: `ModelPtr->GetSkin()`, `ModelPtr->Model.FindPositions_Sequence()`, `glVertexPointer()`, `glTexCoordPointer()`, `glEnableClientState()`, `glDrawElements()`, `LoadModelSkin()`, `FlatBumpTexture()`
- **Notes**: Handles animation with frame interpolation; supports normal and bump mapping in separate texture units

### render_node_object
- **Signature**: `void render_node_object(render_object_data *object, bool other_side_of_media, RenderStep renderStep)`
- **Purpose**: Render game object (sprite/model), handling media boundary clipping
- **Inputs**: Object data, media boundary side flag, render step
- **Outputs**: None
- **Side effects**: Sets GL_CLIP_PLANE5 for media boundary; iterates clipping windows
- **Calls**: `get_polygon_data()`, `get_media_data()`, `glClipPlane()`, `clip_to_window()`, `_render_node_object_helper()`
- **Notes**: Renders in two passes (above/below media) for proper depth ordering through liquid/air boundaries

### _render_node_object_helper
- **Signature**: `void _render_node_object_helper(render_object_data *object, RenderStep renderStep)`
- **Purpose**: Internal helper to render 3D model or sprite
- **Inputs**: Object data, render step
- **Outputs**: None
- **Side effects**: Applies world transforms (translate, rotate, scale); calls RenderModel or sprite rendering
- **Calls**: `glPushMatrix()`, `glTranslated()`, `glRotated()`, `glScalef()`, `RenderModel()`, `setupSpriteTexture()`, `glVertexPointer()`, `glDrawArrays()`
- **Notes**: Disables depth test for non-forced sprites; handles parasitic object depth ordering by y-position

### render_viewer_sprite_layer
- **Signature**: `void render_viewer_sprite_layer(RenderStep renderStep)`
- **Purpose**: Render HUD/weapon sprites in screen space
- **Inputs**: Render step
- **Outputs**: None
- **Side effects**: Saves/restores projection, model, and texture matrices; iterates weapon display queue
- **Calls**: `glMatrixMode()`, `glLoadMatrixd()`, `get_weapon_display_information()`, `render_viewer_sprite()`
- **Notes**: Applies screen-space matrix; handles shape flipping based on shape definition flags

### render_viewer_sprite
- **Signature**: `void render_viewer_sprite(rectangle_definition& RenderRectangle, RenderStep renderStep)`
- **Purpose**: Render individual weapon sprite with screen clipping and glow
- **Inputs**: Rectangle definition, render step
- **Outputs**: None
- **Side effects**: Calculates clipped screen coordinates; builds vertex/texcoord arrays; calls glDrawArrays twice
- **Calls**: `setupSpriteTexture()`, `glVertexPointer()`, `glTexCoordPointer()`, `glDrawArrays()`, `setupGlow()`
- **Notes**: Uses GL_DOUBLE for vertex precision; supports vertical/horizontal flipping; clipping bounds checked against screen

## Control Flow Notes

**Per-frame sequence:**
1. `render_tree()` ΓÇô sets up shader uniforms for landscapes and effects
2. `RenderRasterizerClass::render_tree(kDiffuse)` ΓÇô renders world geometry; calls back to render_node/render_node_side/render_node_floor_or_ceiling for each polygon
3. `render_viewer_sprite_layer(kDiffuse)` ΓÇô renders HUD weapons in diffuse
4. If bloom enabled and no infravision:
   - `blur->begin()` ΓÇô capture glow content to FBO
   - `RenderRasterizerClass::render_tree(kGlow)` ΓÇô re-render with glow shader pass
   - `render_viewer_sprite_layer(kGlow)` ΓÇô render weapon glow
   - `blur->end()`, `blur->draw()` ΓÇô apply bloom postprocessing
5. `render_node_object()` ΓåÆ `_render_node_object_helper()` ΓåÆ `RenderModel()` or sprite rendering for each object

## External Dependencies
- **OpenGL**: `OGL_Headers.h` (GLEW/SDL_opengl), `OGL_Shader.h`, `OGL_FBO.h`, `OGL_Textures.h`
- **Game world**: `lightsource.h`, `media.h`, `player.h`, `weapons.h`, `AnimatedTextures.h`, `ChaseCam.h`, `preferences.h`
- **Base classes**: `RenderRasterize.h` (RenderRasterizerClass), `Rasterizer_Shader.h`
- **External functions** (defined elsewhere): `get_endpoint_data()`, `get_polygon_data()`, `get_media_data()`, `get_light_intensity()`, `View_GetLandscapeOptions()`, `OGL_GetCurrFogData()`, `AnimTxtr_Translate()`, `FindInfravisionVersionRGBA()`, `get_weapon_display_information()`, `LoadModelSkin()`, `FlatBumpTexture()`
- **Globals**: `current_player`, `view`, `cosine_table[]`, `sine_table[]`
