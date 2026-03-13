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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Texture_Configure` | struct | Specifies per-type texture filtering, resolution, color format, and max size |
| `OGL_ConfigureData` | struct | Main configuration container: texture configs, rendering flags, void/landscape colors, anisotropy level, multisampling, sRGB/NPOT/VSync options |
| `OGL_FogData` | struct | Fog parameters: color, depth, start distance, presence flag, landscape affectedness, blend mode, and mode type (Linear/Exp/Exp2) |
| `OGL_ModelData` | class | 3D model preprocessing (scaling, rotation, shifting, sidedness, normals, lighting type) and storage; inherits from `OGL_SkinManager` |
| `OGL_SkinManager` | struct | Manages per-model texture skins (Normal, Glowing, Bump) across bitmap sets; stores OpenGL texture IDs and usage flags |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Using_sRGB` | bool | global | Whether sRGB color space conversion is currently active |
| `Wanting_sRGB` | bool | global | Whether sRGB will be used (preference) |
| `Bloom_sRGB` | bool | global | Whether sRGB framebuffer format is used for bloom effects |
| `FBO_Allowed` | bool | global | Whether framebuffer objects are supported by the driver |
| `npotTextures` | bool | global | Whether non-power-of-two texture dimensions are supported |

## Key Functions / Methods

### OGL_Initialize
- **Purpose:** Initialize OpenGL subsystem and detect available extensions.
- **Inputs:** None.
- **Outputs/Return:** `bool` ΓÇô true if OpenGL is available and initialized.
- **Side effects:** Global state initialization; extension detection.
- **Calls:** (Not visible in header.)
- **Notes:** Precursor to any OpenGL usage.

### OGL_IsPresent / OGL_IsActive
- **Purpose:** Query OpenGL availability and activation status.
- **Inputs:** None.
- **Outputs/Return:** `bool` ΓÇô presence or active status.
- **Side effects:** None.
- **Calls:** None visible.

### OGL_CheckExtension
- **Purpose:** Test for a named OpenGL extension (e.g., `"GL_ARB_texture_compression"`).
- **Inputs:** `const std::string` ΓÇô extension name.
- **Outputs/Return:** `bool` ΓÇô whether extension is available.
- **Side effects:** None.
- **Calls:** None visible.

### sRGB_frob (inline)
- **Purpose:** Convert linear float color component to/from sRGB using standard sRGB spec curve.
- **Inputs:** `GLfloat f` ΓÇô color channel value (typically 0ΓÇô1).
- **Outputs/Return:** `GLfloat` ΓÇô converted value; unchanged if `Using_sRGB` is false.
- **Side effects:** None.
- **Calls:** `std::pow()` for gamma correction.
- **Notes:** Uses EXT_framebuffer_sRGB reference spec; implements exact piecewise formula.

### SglColor* (color wrapper functions)
- **Purpose:** Wrapper around OpenGL color calls; applies sRGB conversion if enabled.
- **Variants:** `SglColor3f`, `SglColor4f`, `SglColor3ub`, `SglColor3us`, etc.
- **Inputs:** Color components (float or unsigned int types).
- **Outputs/Return:** None.
- **Side effects:** OpenGL color state modified.
- **Notes:** Conditional compilation under `HAVE_OPENGL`.

### OGL_StartProgress / OGL_ProgressCallback / OGL_StopProgress
- **Purpose:** Frame progress indication during long operations (texture/model loading).
- **Inputs:** `OGL_StartProgress(int total)`, `OGL_ProgressCallback(int delta)`.
- **Outputs/Return:** None.
- **Side effects:** UI progress update.
- **Notes:** Progress tracking for user feedback.

### OGL_CountTextures / OGL_LoadTextures / OGL_UnloadTextures
- **Purpose:** Resource count and lifecycle management for textures by collection.
- **Inputs:** `short Collection` ΓÇô collection ID.
- **Outputs/Return:** Count (int) or void.
- **Side effects:** Texture VRAM allocation/deallocation.
- **Calls:** Implementation in separate file.

### OGL_CountModelsImages / OGL_LoadModelsImages / OGL_UnloadModelsImages
- **Purpose:** Resource count and lifecycle management for 3D models and skins by collection.
- **Inputs:** `short Collection` ΓÇô collection ID.
- **Outputs/Return:** Count (int) or void.
- **Side effects:** Model VRAM allocation/deallocation.
- **Calls:** Implementation in separate file.

### OGL_ResetTextures
- **Purpose:** Clear all loaded textures and force reload from disk.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** All textures unloaded; next render pass will reload.
- **Notes:** Implemented in `OGL_Textures.cpp`.

### OGL_GetFogData
- **Purpose:** Retrieve fog configuration by type (AboveLiquid or BelowLiquid).
- **Inputs:** `int Type` ΓÇô fog type constant.
- **Outputs/Return:** `OGL_FogData*` ΓÇô pointer to fog config; NULL if not found.
- **Side effects:** None.

### Get_OGL_ConfigureData
- **Purpose:** Access current OpenGL configuration data.
- **Inputs:** None.
- **Outputs/Return:** `OGL_ConfigureData&` ΓÇô reference to active configuration.
- **Side effects:** None.
- **Notes:** Used by renderers to query texture/model quality settings and flags.

### parse_mml_opengl / reset_mml_opengl
- **Purpose:** Parse MML configuration files to override default OpenGL settings; reset to defaults.
- **Inputs:** `const InfoTree& root` ΓÇô parsed MML tree.
- **Outputs/Return:** None.
- **Side effects:** Modifies global configuration.
- **Notes:** Enables modding via external XML-like markup files.

## Control Flow Notes
This is a **configuration and initialization** header, not a per-frame update mechanism. The typical flow:
1. **Initialization:** `OGL_Initialize()` detects and sets up OpenGL.
2. **Configuration:** `Get_OGL_ConfigureData()` and MML parsing populate settings.
3. **Resource Load:** `OGL_LoadTextures()`, `OGL_LoadModels()` allocate VRAM based on config.
4. **Render Loop:** Rendering code queries config via `Get_OGL_ConfigureData()` and uses flags like `OGL_Flag_Fog`, `OGL_Flag_3D_Models`, etc.
5. **Reload:** `OGL_ResetTextures()` can be called dynamically to clear and reload if textures corrupt.

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
