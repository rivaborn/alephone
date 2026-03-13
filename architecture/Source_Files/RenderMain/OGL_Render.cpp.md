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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SurfaceCoords` | struct | Texture coordinate vectors (U, V, W) and complement vectors for surface mapping |
| `ShaderDataStruct` | struct | Model, skin, color, and collection data passed to shader callbacks |
| `LightingDataStruct` | struct | Per-vertex lighting parameters: light type, direction, intensity, color planes |
| `OGL_FogData` | struct | Fog configuration; defined elsewhere, pointer stored in `CurrFog` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_OGL_IsActive` | `static bool` | static | Whether OpenGL is currently active for rendering |
| `JustInited` | `static bool` | static | Flag to force viewport/matrix update after context creation |
| `SavedScreenBounds`, `SavedViewBounds` | `static Rect` | static | Cached viewport bounds; detect changes to trigger recomputation |
| `ViewWidth`, `ViewHeight` | `short` (global) | global | Current view dimensions in pixels |
| `CenteredWorld_2_MaraEye`, `World_2_MaraEye`, `World_2_OGLEye`, `CenteredWorld_2_OGLEye` | `static GLdouble[16]` | static | Transformation matrices for coordinate conversion |
| `OGLEye_2_Clip`, `OGLEye_2_Screen`, `Screen_2_Clip` | `static GLdouble[16]` | static | Projection and screen-space transformation matrices |
| `ProjectionType` | `static int` | static | Currently-loaded projection matrix (cache to avoid redundant GL calls) |
| `XScale`, `YScale`, `XOffset`, `YOffset`, `XScaleRecip`, `YScaleRecip` | `static GLdouble` | static | Screen-to-world and world-to-screen scaling factors |
| `StaticPatterns[4][32]` | `static GLuint` | static | Precomputed stipple patterns for static effect (4 channels ├ù 32 dwords) |
| `UseFlatStatic` | `static bool` | static | Whether to render static as flat color vs. stipple pattern |
| `FlatStaticColor[4]` | `static uint16` | static | RGBA color for flat static effect |
| `StaticRandom` | `static GM_Random` | static | Random number generator for static pattern variance |
| `Yaw` | `static double` | static | Current camera yaw angle (full circle = 1) |
| `LandscapeRescale` | `static double` | static | Scaling to match software renderer landscape appearance |
| `SelfLuminosity` | `static _fixed` | static | Miner's light effect intensity |
| `CurrFog`, `CurrFogColor[4]` | `static OGL_FogData*`, `static GLfloat[4]` | static | Current fog parameters and RGBA color; can differ from stored fog due to infravision |
| `BlendType` | `static short` | static | Currently-active blend mode; cached to skip redundant GL calls |
| `HorizCoords`, `VertCoords` | `static SurfaceCoords` | static | Texture coordinate systems for horizontal and vertical surfaces |

## Key Functions / Methods

### OGL_StartRun
- **Signature:** `bool OGL_StartRun()`
- **Purpose:** Initialize OpenGL rendering context, check extensions, allocate textures and shaders.
- **Inputs:** None (reads from preferences and engine state)
- **Outputs/Return:** `true` if successful; `false` if preconditions not met or GL extension missing
- **Side effects:** Modifies global GL state, calls `glEnable`, `glDisable`, `glBlendFunc`, etc.; initializes texture manager, font rendering, model shaders
- **Calls:** `OGL_IsPresent`, `glewInit` (Win32), `OGL_CheckExtension`, `OGL_StartTextures`, `PreloadTextures`, `SetupShaders`, `OGL_ResetModelSkins`, `FontSpecifier::OGL_ResetFonts`
- **Notes:** Loads replacement texture collections; handles sRGB and NPOT extension availability; sets up Z-buffering, culling, alpha-test, and vertex array states.

### OGL_StopRun
- **Signature:** `bool OGL_StopRun()`
- **Purpose:** Clean up and deactivate OpenGL rendering.
- **Calls:** `OGL_StopTextures`, `Shader::unloadAll`
- **Notes:** Inverse of `OGL_StartRun`; safe to call if not active.

### OGL_StartMain
- **Signature:** `bool OGL_StartMain()`
- **Purpose:** Configure rendering state for the current frame: fog setup, void color, depth clear.
- **Side effects:** Calls `glEnable(GL_FOG)` / `glDisable`, `glFogf`, `glClearColor`, `glClear`, `glEnable(GL_FRAMEBUFFER_SRGB_EXT)` if sRGB is wanted
- **Calls:** `OGL_GetFogData`, `FogActive`, `glFogfv`, `IsInfravisionActive`, `FindInfravisionVersionRGBA`, randomizes static patterns
- **Notes:** One-time per-frame setup; fog parameters read from stored data or changed on-the-fly.

### OGL_EndMain
- **Signature:** `bool OGL_EndMain()`
- **Purpose:** Finalize frame rendering: apply faders, reset projection and modelview matrices.
- **Calls:** `SetProjectionType(Projection_Screen)`, `glDisable(GL_DEPTH_TEST)`, `OGL_DoFades`
- **Notes:** Disables depth test and texture before faders; restores default render state.

### OGL_SetWindow
- **Signature:** `bool OGL_SetWindow(Rect &ScreenBounds, Rect &ViewBounds, bool UseBackBuffer)`
- **Purpose:** Update viewport dimensions and recompute projection matrices when window bounds change.
- **Inputs:** Screen bounds (full display), view bounds (rendering area), back-buffer flag
- **Side effects:** Modifies `ViewWidth`, `ViewHeight`, `Screen_2_Clip` matrix, sets `ProjectionType` to `Projection_NONE` to force reload
- **Calls:** `glMatrixMode`, `glGetDoublev`, `RectsEqual`
- **Notes:** Skips updates if bounds unchanged and not just-initialized.

### PreloadTextures
- **Signature:** `void PreloadTextures()`
- **Purpose:** Scan map and preload all used wall/floor/ceiling textures to avoid stalls during gameplay.
- **Side effects:** Iterates all polygons and sides; uploads textures to GPU
- **Calls:** `PreloadWallTexture` via `for_each` on texture set
- **Notes:** Uses `std::set<TextureWithTransferMode>` to avoid duplicate preloads; reduces lazy-loading overhead.

### PreloadWallTexture
- **Signature:** `void PreloadWallTexture(const TextureWithTransferMode& inTexture)`
- **Purpose:** Upload a single texture variant (with transfer mode) to GPU.
- **Inputs:** Shape descriptor and transfer mode pair
- **Side effects:** Calls `TextureManager` to load and render texture
- **Calls:** `AnimTxtr_Translate`, `get_shape_bitmap_and_shading_table`, `TextureManager::Setup`, `TextureManager::RenderNormal`, `RenderGlowing`, `RenderBump`
- **Notes:** Handles landscapes specially (applies vertical repeat and aspect-ratio options); skips UNONE textures; infravision inactive at load time.

### RenderModelSetup
- **Signature:** `static bool RenderModelSetup(rectangle_definition& RenderRectangle)`
- **Purpose:** Configure matrices, clipping planes, and transformations for 3D sprite/model rendering.
- **Outputs/Return:** `true` if model could be rendered; calls `RenderModel` internally
- **Side effects:** `glPushMatrix`, `glLoadMatrixd`, `glTranslatef`, `glRotated`, `glScalef`, enables/disables clip planes
- **Calls:** `glClipPlane` (up to 5 planes for left/right/top/bottom/liquid boundaries)
- **Notes:** Handles animation frame blending; applies liquid clipping if model straddles media boundary.

### RenderModel
- **Signature:** `static bool RenderModel(rectangle_definition& RenderRectangle, short Collection, short CLUT)`
- **Purpose:** Render a 3D sprite with textures, lighting, and blending.
- **Inputs:** Render rectangle (position, size, orientation), collection and CLUT
- **Outputs/Return:** `true` if skin found; `false` if rendering failed
- **Side effects:** Binds textures, sets blend function, enables/disables culling, configures lighting arrays
- **Calls:** `ModelPtr->GetSkin`, `DoLightingAndBlending`, `SetupStaticMode`, `TeardownStaticMode`, `ModelRenderObject.Render`, `SetBlend`, `glCullFace`
- **Notes:** Handles three transfer modes: static (with stipple/flat/stencil), tinted (invisibility), normal; applies external lighting callback for miner's-light effect.

### DoLightingAndBlending
- **Signature:** `static bool DoLightingAndBlending(rectangle_definition& RenderRectangle, bool& IsBlended, GLfloat *Color, bool& ExternallyLit)`
- **Purpose:** Determine lighting color, blending mode, and glowmap eligibility based on transfer mode and environment.
- **Inputs:** Rectangle definition, flags
- **Outputs:** `IsBlended`, `Color[4]` (RGBA), `ExternallyLit` flag
- **Returns:** `true` if glowmapping is possible; `false` for static, tinted, infravision
- **Side effects:** `glEnable/Disable GL_BLEND`, `GL_ALPHA_TEST`
- **Notes:** Maps engine transfer modes (_static, _tinted, _shadeless) to OpenGL states; handles invisibility blending.

### SetupStaticMode / TeardownStaticMode
- **Signature:** `void SetupStaticMode(int16 transfer_data)` / `void TeardownStaticMode()`
- **Purpose:** Configure or disable static effect rendering (noise overlay for invisibility/special effects).
- **Side effects (Setup):** If `UseFlatStatic`: `glEnable(GL_BLEND)`, sets flat color. Else: generates random stipple patterns or sets up stencil buffer.
- **Calls (Setup):** `StaticRandom.KISS()`, `StaticRandom.LFIB4()`, `glEnable(GL_POLYGON_STIPPLE)` or stencil calls
- **Side effects (Teardown):** Disables stipple or stencil; restores blend function
- **Notes:** Two implementations: stipple-based (fast, aliased) and stencil-based (slower, per-triangle). Opacity controlled by `transfer_data`.

### OGL_RenderCrosshairs
- **Signature:** `bool OGL_RenderCrosshairs()`
- **Purpose:** Render crosshair UI overlay at center of screen.
- **Side effects:** `SetProjectionType(Projection_Screen)`, `glTranslated`, `glRotated` per quadrant, calls `OGL_RenderRect`
- **Calls:** `GetCrosshairData`, `SetProjectionType`
- **Notes:** Supports circle and real-crosshairs shapes; rotates 4 times (quadrants); honors thickness and color from preferences.

### OGL_RenderText
- **Signature:** `bool OGL_RenderText(short BaseX, short BaseY, const char *Text, unsigned char r, unsigned char g, unsigned char b)`
- **Purpose:** Render a text string at given screen position with shadow and color.
- **Side effects:** Creates display list, calls `glNewList`, `glCallList`, `glDeleteLists`
- **Calls:** `SetProjectionType`, `glMatrixMode`, `glLoadIdentity`, `glTranslatef`, `GetOnScreenFont().OGL_Render`
- **Notes:** Draws black shadow at offset, then color text; uses display lists for reusability.

### OGL_RenderRect / OGL_RenderTexturedRect / OGL_RenderFrame / OGL_RenderLines
- **Signature:** `void OGL_RenderRect(float x, float y, float w, float h)` and variants
- **Purpose:** Render 2D geometric primitives (rectangles, textured quads, frames, lines).
- **Side effects:** Temporarily disables `GL_TEXTURE_2D` and texture-coord arrays, sets `glVertexPointer`, calls `glDrawArrays`
- **Notes:** Used for UI and debug overlays; simple immediate-mode rendering using vertex arrays.

### LightingCallback
- **Signature:** `static void LightingCallback(void *Data, size_t NumVerts, GLfloat *Normals, GLfloat *Positions, GLfloat *Colors)`
- **Purpose:** Shader callback for per-vertex lighting calculation in model rendering.
- **Inputs:** Lighting data struct, vertex count, normal vectors, world positions
- **Outputs:** Computed RGBA colors per vertex
- **Calls:** `FindShadingColor` (for horizon-based lighting)
- **Notes:** Two light modes: fast (directional + miner's light) and individual (per-vertex depth-based). Handles semitransparency (opacity multiplied into alpha).

### SetBlend
- **Signature:** `static void SetBlend(short _BlendType)`
- **Purpose:** Set OpenGL blending function, caching to avoid redundant calls.
- **Inputs:** Blend mode enum (_crossfade, _add, _premult variants)
- **Side effects:** `glBlendFunc` called only if blend type changes
- **Notes:** Reduces state change overhead.

### SurfaceCoords::FindComplements
- **Signature:** `bool SurfaceCoords::FindComplements()`
- **Purpose:** Calculate 3D complement vectors for texture coordinate system.
- **Outputs:** Populates `U_CmplVec`, `V_CmplVec`, `W_CmplVec`
- **Returns:** `true` if U and V vectors non-collinear; `false` otherwise
- **Calls:** `ScalarProd`, `VecScalarMult`, `VecAdd`, `VectorProd`
- **Notes:** Used internally for texture mapping; assumes orthogonal vectors (Marathon convention) but code general.

## Control Flow Notes
1. **Initialization phase** (`OGL_StartRun`): Creates GL context, loads extensions, preloads textures, initializes shaders.
2. **Frame start** (`OGL_StartMain`): Clears depth/color, configures fog, randomizes static patterns.
3. **Rendering** (called by engine): Models/sprites rendered via `RenderModelSetup` ΓåÆ `RenderModel`, UI rendered via `OGL_RenderCrosshairs`, `OGL_RenderText`, etc.
4. **Frame end** (`OGL_EndMain`): Applies post-process faders, resets projection/modelview.
5. **Shutdown** (`OGL_StopRun`): Unloads textures, shaders, deactivates GL.

Coordinate pipelines:
- **World** ΓåÆ **Marathon eye** (rotated, translation offset) ΓåÆ **OpenGL eye** (rotated axes) ΓåÆ **Clip space** (perspective projection)
- **Screen** (2D x/y in pixels) ΓåÆ **Ray in OpenGL eye space** (for picking)
- **Texture coords** ΓåÆ **Complement vectors** (for surface intersection calculations)

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
