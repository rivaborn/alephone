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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TOKey` | typedef (pair) | Hash key for texture lookup: `(CLUT, Bitmap)` |
| `TOHash` | typedef (unordered_map) | Hash map from TOKey to OGL_TextureOptions |
| `OGL_TextureOptions` | struct | Rendering configuration (opacity, blending, bloom, dimension overrides, billboard mode) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DefaultTextureOptions` | `OGL_TextureOptions` | static | Fallback texture options when no specific config exists |
| `Collections` | `TOHash[NUMBER_OF_COLLECTIONS]` | static | Array of hash maps storing all texture options per collection |

## Key Functions / Methods

### OGL_GetTextureOptions
- **Signature:** `OGL_TextureOptions *OGL_GetTextureOptions(short Collection, short CLUT, short Bitmap)`
- **Purpose:** Retrieve texture options with cascading fallback
- **Inputs:** Collection ID, CLUT index, Bitmap index
- **Outputs/Return:** Non-null pointer to OGL_TextureOptions (always succeeds)
- **Side effects:** None (read-only)
- **Calls:** `IsInfravisionTable()`, `IsSilhouetteTable()` (external)
- **Notes:** Fallback chain: exact (CLUT, Bitmap) ΓåÆ infravision variant ΓåÆ silhouette variant ΓåÆ (ALL_CLUTS, Bitmap) ΓåÆ default

### OGL_LoadTextures
- **Signature:** `void OGL_LoadTextures(short Collection)`
- **Purpose:** Load all textures in a collection into GPU memory
- **Inputs:** Collection ID
- **Outputs/Return:** void
- **Side effects:** GPU texture allocation; calls progress callback per texture
- **Calls:** `OGL_TextureOptions::Load()`, `OGL_ProgressCallback(int)` (extern)
- **Notes:** Likely used during level load

### OGL_UnloadTextures
- **Signature:** `void OGL_UnloadTextures(short Collection)`
- **Purpose:** Unload all textures in a collection from GPU
- **Inputs:** Collection ID
- **Outputs/Return:** void
- **Side effects:** GPU texture deallocation
- **Calls:** `OGL_TextureOptions::Unload()`

### parse_mml_opengl_texture
- **Signature:** `void parse_mml_opengl_texture(const InfoTree& root)`
- **Purpose:** Parse a single texture definition from MML config
- **Inputs:** InfoTree node with `coll`, `bitmap`, `clut`, `clut_variant`, and ~20 texture properties
- **Outputs/Return:** void
- **Side effects:** Creates or updates entries in `Collections[coll]` hash map
- **Calls:** `InfoTree::read_indexed()`, `InfoTree::read_attr()`, `InfoTree::read_path()`
- **Notes:** 
  - Translates deprecated CLUT values to variant system
  - Loops over CLUT_VARIANT range if "all variants" mode requested
  - Derives internal CLUT ID based on variant (infravision/silhouette + CLUT-specific or global)
  - Reads opacity, blending, bloom, image paths, masks, dimensions, billboard mode, etc.

### parse_mml_opengl_txtr_clear
- **Signature:** `void parse_mml_opengl_txtr_clear(const InfoTree& root)`
- **Purpose:** Clear texture definitions
- **Inputs:** InfoTree with optional `coll` attribute
- **Outputs/Return:** void
- **Side effects:** Clears one or all Collections hash maps
- **Calls:** `TODelete()`, `TODelete_All()`

## Control Flow Notes
1. **Init:** `reset_mml_opengl_texture()` clears all collections
2. **Config parse:** MML parser calls `parse_mml_opengl_texture()` for each texture element
3. **Runtime:** `OGL_GetTextureOptions()` queried during rendering to fetch texture settings
4. **GPU lifecycle:** `OGL_LoadTextures()` called on level load; `OGL_UnloadTextures()` on unload
5. Frame/render integration: not inferable from this file

## External Dependencies
- `cseries.h`: Platform abstraction, type definitions (int16, short)
- `OGL_Subst_Texture_Def.h`: Module header; `OGL_TextureOptions` struct definition
- `Logging.h`: Logging macros (included but unused here)
- `InfoTree.h`: XML/INI config parser for MML loading
- `<boost/unordered_map.hpp>`: Hash map container
- `OGL_ProgressCallback(int)` ΓÇö defined elsewhere; progress reporting
- `IsInfravisionTable()`, `IsSilhouetteTable()` ΓÇö defined elsewhere; CLUT type checks
- `OGL_TextureOptions::Load()`, `Unload()` ΓÇö defined in OGL_Texture_Def.h
