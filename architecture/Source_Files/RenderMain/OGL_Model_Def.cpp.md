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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SequenceMapEntry` | struct | Maps a Marathon physics sequence ID to a model animation sequence ID |
| `ModelDataEntry` | struct | Container for model geometry, sequence mappings, and transformation parameters |
| `ModelHashEntry` | struct | Hash table entry storing model index and sequence table offset |
| `OGL_SkinData` | struct | Texture configuration per CLUT (color lookup table) with opacity/blending/bloom settings |
| `OGL_SkinManager` | class | Manages multiple skins per model; holds OpenGL texture IDs and load state |
| `OGL_ModelData` | class | Complete model including file path, transformations (rotation/scale/shift), normals, lighting, and skin manager |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DefaultModelData` | `OGL_ModelData` | static | Template for initializing new model entries |
| `DefaultSkinData` | `OGL_SkinData` | static | Template for initializing new skin entries |
| `MdlList[NUMBER_OF_COLLECTIONS]` | `vector<ModelDataEntry>` | static | Stores all model definitions per collection ID |
| `MdlHash[NUMBER_OF_COLLECTIONS]` | `vector<ModelHashEntry>` | static | Hash table for O(1) model lookup by sequence; resized on-demand to `MdlHashSize=256` |
| `ForcingSpriteDepth` | bool | static | Global flag controlling sprite/model depth-sort interaction |

## Key Functions / Methods

### OGL_GetModelData
- **Signature:** `OGL_ModelData *OGL_GetModelData(short Collection, short Sequence, short& ModelSequence)`
- **Purpose:** Primary lookup function; returns model data for a given collection and Marathon sequence number.
- **Inputs:** Collection ID, Marathon animation sequence ID; outputs mapped model sequence via reference.
- **Outputs/Return:** Pointer to `OGL_ModelData` if found; NULL otherwise. Sets `ModelSequence` to the corresponding model sequence (or NONE).
- **Side effects:** Initializes hash table on first call per collection; updates hash entries on misses.
- **Calls:** (none visible in this file; internal iteration and comparison)
- **Notes:** Uses two-level lookup: (1) hash table hit ΓåÆ sequence-map check ΓåÆ neutral sequence check; (2) hash miss ΓåÆ linear search through `MdlList[Collection]`, then update hash. Hash collisions are resolved by linear probing in the fallback.

### OGL_ModelData::Load
- **Signature:** `void OGL_ModelData::Load()`
- **Purpose:** Loads model geometry from disk, applies transformations, and computes bounding boxes.
- **Inputs:** (none; uses member variables: `ModelFile`, `ModelType`, rotation/scale/shift parameters)
- **Outputs/Return:** (none; modifies `Model` member and skin data)
- **Side effects:** Clears and populates `Model` (3D geometry); calls loader functions; applies 3├ù3 rotation + scale matrices; shifts bounding box. Calls `OGL_SkinManager::Load()` to load skins.
- **Calls:** `LoadModel_Wavefront`, `LoadModel_Studio`, `LoadModel_Dim3`, `LoadModel_QD3D` (and RightHand variants); matrix functions `MatIdentity`, `MatMult`, `MatScalMult`, `MatVecMult`; `Model.FindPositions_Neutral()`, `Model.FindBoundingBox()`, `Model.AdjustNormals()`, `Model.CalculateTangents()`.
- **Notes:** Distinguishes animated (uses transformation matrix stored in `Model.TransformPos/Norm`) vs. static (applies transformation in-place to vertex positions and normals). Inverts normal scaling if `Scale < 0` (mirroring). Coordinates system conversions for OBJ and MAX formats are handled by calling RightHand variants.

### OGL_ModelData::Unload
- **Signature:** `void OGL_ModelData::Unload()`
- **Purpose:** Releases model geometry and associated resources.
- **Inputs:** (none)
- **Outputs/Return:** (none)
- **Side effects:** Clears `Model`; resets `ForcingSpriteDepth` flag; calls `OGL_SkinManager::Unload()`.
- **Calls:** `Model.Clear()`, `OGL_ResetForceSpriteDepth()`, `OGL_SkinManager::Unload()`.

### OGL_SkinManager::GetSkin
- **Signature:** `OGL_SkinData *OGL_SkinManager::GetSkin(short CLUT)`
- **Purpose:** Retrieves skin data matching a requested CLUT (color lookup table), with fallback logic for infravision and silhouette variants.
- **Inputs:** CLUT ID (color table identifier)
- **Outputs/Return:** Pointer to `OGL_SkinData` if found; NULL otherwise.
- **Side effects:** (none)
- **Calls:** `IsInfravisionTable()`, `IsSilhouetteTable()` (defined elsewhere).
- **Notes:** Priority order: exact CLUT match ΓåÆ infravision fallback ΓåÆ silhouette fallback ΓåÆ `ALL_CLUTS` wildcard. Multiple loops suggest inefficiency; could be optimized.

### OGL_SkinManager::Use
- **Signature:** `bool OGL_SkinManager::Use(short CLUT, short Which)`
- **Purpose:** Prepares a skin texture for rendering; allocates OpenGL texture ID if not yet allocated.
- **Inputs:** CLUT and texture type (Normal/Glowing/Bump).
- **Outputs/Return:** Returns `true` if texture needs loading (newly allocated); `false` if already in use.
- **Side effects:** Calls `glGenTextures()` and `glBindTexture()` on first use; sets `IDsInUse` and `IDs` array entries.
- **Calls:** `glGenTextures()`, `glBindTexture()`.
- **Notes:** Assumes OpenGL context is active. ID generation and binding are split; actual texture data loading deferred to caller.

### parse_mml_opengl_model
- **Signature:** `void parse_mml_opengl_model(const InfoTree& root)`
- **Purpose:** Parses XML/MML model configuration and registers into `MdlList`.
- **Inputs:** `InfoTree` XML node containing model definition and nested `<seq_map>` and `<skin>` elements.
- **Outputs/Return:** (none)
- **Side effects:** Adds or updates entry in `MdlList[Collection]`; modifies global model list.
- **Calls:** `root.read_*()` methods; `std::sort()` for sequence map comparison.
- **Notes:** Reads collection, sequence, files, transformations, normal type, lighting type, and skins. Handles deprecated CLUT options (INFRAVISION/SILHOUETTE) by translating to `ALL_CLUTS` + `clut_variant`. Avoids duplicates by searching for existing entry with matching sequence and sequence-map before inserting.

### Helper matrix functions
- **MatCopy, MatIdentity, MatMult, MatScalMult, MatVecMult:** Static 3├ù3 matrix utilities for rotation/scaling transformations. All operate on `GLfloat` arrays. `MatVecMult` applies transformation to a position vector.
- **StringsEqual:** Case-insensitive string comparison up to max length; used to identify model file types.

## Control Flow Notes
- **Load phase:** `OGL_LoadModels(Collection)` iterates all entries in `MdlList[Collection]`, calling `Load()` on each and setting `ForcingSpriteDepth` if any model requests it.
- **Lookup:** `OGL_GetModelData()` is called at render time to retrieve model data for a given sequence; hash table accelerates repeated lookups of the same sequence.
- **Configuration:** `parse_mml_opengl_model()` is called during engine initialization/MML parsing to populate the model database; `reset_mml_opengl_model()` and `parse_mml_opengl_model_clear()` clear entries.
- **Unload:** `OGL_UnloadModels()` and `OGL_ResetModelSkins()` clean up on shutdown or texture reset.

## External Dependencies
- **Includes:** `cseries.h` (Marathon types), `OGL_Setup.h`, `OGL_Model_Def.h`, `Dim3_Loader.h`, `StudioLoader.h`, `WavefrontLoader.h`, `InfoTree.h`
- **External functions:** `LoadModel_Wavefront`, `LoadModel_Studio`, `LoadModel_Dim3`, `LoadModel_QD3D` (model loaders); `OGL_ProgressCallback`, `Get_OGL_ConfigureData` (OpenGL setup)
- **External types:** `FileSpecifier`, `Model3D`, `InfoTree`, `OGL_SkinData`, OpenGL (GL*).
- **Defined elsewhere:** `NONE`, `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`, `ALL_CLUTS`, `INFRAVISION_BITMAP_SET`, `SILHOUETTE_BITMAP_SET`, infravision/silhouette helper predicates.
