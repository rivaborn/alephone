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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_SkinData` | struct | Per-CLUT skin definition; inherits texture options (opacity, blending, bloom) |
| `OGL_SkinManager` | struct | Container for skins; tracks OpenGL texture IDs and usage state for Normal/Glowing/Bump variants |
| `OGL_ModelData` | class | Complete model definition: geometry (`Model3D`), transformation parameters (scale, rotation, shift), lighting/depth type, and embedded skin manager |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `OGL_MLight_Fast`, `OGL_MLight_Fast_NoFade`, `OGL_MLight_Indiv`, `OGL_MLight_Indiv_NoFade` | enum constants | global | Lighting calculation modes; fast (one light) vs. per-vertex, with optional fade toward sides |
| `NUMBER_OF_MODEL_LIGHT_TYPES` | enum constant | global | Count of lighting mode variants |

## Key Functions / Methods

### OGL_SkinManager::Reset()
- Signature: `void Reset(bool Clear_OGL_Txtrs)`
- Purpose: Reset skin data in preparation for reloading; optionally deallocate OpenGL textures
- Inputs: `Clear_OGL_Txtrs` ΓÇô whether to clear GPU texture objects
- Outputs/Return: None
- Side effects: Clears `SkinData` vector; may deallocate GPU memory
- Calls: (defined elsewhere)
- Notes: Must be called before reloading skins from disk

### OGL_SkinManager::GetSkin()
- Signature: `OGL_SkinData *GetSkin(short CLUT)`
- Purpose: Retrieve skin data for a specific color lookup table
- Inputs: `CLUT` ΓÇô color table ID (or `ALL_CLUTS` for default)
- Outputs/Return: Pointer to `OGL_SkinData`, or NULL if not available
- Side effects: None
- Calls: Vector indexing on `SkinData`
- Notes: Returns NULL for missing CLUTs; safe to call any time

### OGL_SkinManager::Use()
- Signature: `bool Use(short CLUT, short Which)`
- Purpose: Activate a texture variant and determine if it needs loading
- Inputs: `CLUT` ΓÇô color table ID; `Which` ΓÇô texture enum (Normal/Glowing/Bump)
- Outputs/Return: `true` if texture should be loaded, `false` if already in use
- Side effects: Updates `IDsInUse` tracking array
- Calls: (defined elsewhere)
- Notes: Manages texture ID lifecycle

### OGL_ModelData::ModelPresent()
- Signature: `bool ModelPresent() {return !Model.VertIndices.empty();}`
- Purpose: Check whether 3D geometry has been loaded
- Inputs: None
- Outputs/Return: `true` if vertex data exists
- Side effects: None
- Calls: `Model3D::VertIndices.empty()`
- Notes: Inline; read-only check

### OGL_GetModelData()
- Signature: `OGL_ModelData *OGL_GetModelData(short Collection, short Sequence, short& ModelSequence)`
- Purpose: Look up model for a game collection and sequence pair
- Inputs: `Collection` ID, `Sequence` ID within collection
- Outputs/Return: Pointer to associated `OGL_ModelData`, or NULL; output param `ModelSequence` gives actual model sequence used
- Side effects: None
- Calls: (defined elsewhere)
- Notes: Handles multiple sequences mapping to single model

### OGL_CountModels() / OGL_LoadModels() / OGL_UnloadModels()
- Signature: `int OGL_CountModels(short Collection)`, `void OGL_LoadModels(short Collection)`, `void OGL_UnloadModels(short Collection)`
- Purpose: Batch collection-level model resource management
- Inputs: `Collection` ΓÇô collection ID
- Outputs/Return: Count of models (CountModels only)
- Side effects: File I/O and GPU memory allocation/deallocation
- Calls: (defined elsewhere)
- Notes: **Important:** Must call `OGL_ResetForceSpriteDepth()` before `OGL_LoadModels()` to clear sprite depth state

**Remaining functions** (`OGL_ResetModelSkins`, `OGL_ResetForceSpriteDepth`, `OGL_ForceSpriteDepth`, MML parsers): Global state management and data-driven configuration. See function signatures inline.

## Control Flow Notes
This is a declaration header; implementation is elsewhere. Typical workflow:
1. **Init**: `parse_mml_opengl_model()` populates global model registry from MML data
2. **Load**: `OGL_LoadModels(collection)` iterates over collection's models and calls `OGL_ModelData::Load()` for each
3. **Runtime**: Rendering code queries `OGL_GetModelData()` to retrieve model/skin data
4. **Shutdown**: `OGL_UnloadModels()` and `OGL_ResetModelSkins()` clean up resources

Does not directly participate in frame/render loops; used during level load and resource transitions.

## External Dependencies
- `OGL_Texture_Def.h`: `OGL_TextureOptionsBase` (base class), texture enums, `MAXIMUM_CLUTS_PER_COLLECTION`, `NUMBER_OF_OPENGL_BITMAP_SETS`
- `Model3D.h`: `Model3D` struct (geometry storage)
- `OGL_Headers.h`: OpenGL types/functions (`GLuint`, `GLushort`, etc.)
- STL: `vector<>`
- Elsewhere: `FileSpecifier`, `ImageDescriptor`, `InfoTree`
