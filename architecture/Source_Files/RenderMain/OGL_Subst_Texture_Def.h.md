# Source_Files/RenderMain/OGL_Subst_Texture_Def.h

## File Purpose
Header file defining substitute texture configuration and management for OpenGL-rendered walls and sprites in the Aleph One game engine. Extends base texture definitions with sprite-specific billboard modes and texture loading/unloading infrastructure.

## Core Responsibilities
- Define `OGL_TextureOptions` structure for wall/sprite substitute textures
- Provide billboard type enumeration for sprite rendering modes
- Declare texture collection management functions (load/unload/count)
- Declare MML (configuration) parsing and reset functions for texture definitions
- Query current texture options by collection, CLUT, and bitmap ID

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `BillboardType` | `enum class` | Sprite orientation modes: User-defined, Y-axis aligned, XY-plane aligned |
| `OGL_TextureOptions` | `struct` | Extends `OGL_TextureOptionsBase` with void visibility, tile ratio scaling, and billboard type for sprites |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_GetTextureOptions
- **Signature:** `OGL_TextureOptions *OGL_GetTextureOptions(short Collection, short CLUT, short Bitmap);`
- **Purpose:** Retrieve the active texture configuration for a specific bitmap in a collection
- **Inputs:** Collection ID, CLUT (color lookup table), Bitmap ID
- **Outputs/Return:** Pointer to `OGL_TextureOptions` struct
- **Side effects:** None inferable from signature
- **Calls:** Not visible (declaration only)
- **Notes:** Used to query texture options at render time

### OGL_CountTextures, OGL_LoadTextures, OGL_UnloadTextures
- **Purpose:** Resource management for texture collections (count, load into GPU memory, unload)
- **Inputs:** `short Collection` (collection ID)
- **Outputs/Return:** `int` count for `OGL_CountTextures`; void for load/unload
- **Side effects:** GPU memory allocation/deallocation, texture binding
- **Calls:** Not visible (declarations only)
- **Notes:** Batch operations on entire collections

### parse_mml_opengl_texture, reset_mml_opengl_texture, parse_mml_opengl_txtr_clear
- **Purpose:** MML (configuration file) parsing for texture definitions, and reset/clear operations
- **Inputs:** `const InfoTree& root` (configuration tree node)
- **Outputs/Return:** Void
- **Side effects:** Updates global texture configuration state
- **Calls:** Not visible (declarations only)
- **Notes:** Likely invoked during engine initialization or hot-reload

## Control Flow Notes
This file is part of the texture initialization and asset management pipeline. Textures are loaded per collection (likely at level/scene load), with per-bitmap options queryable at render time. MML parsing enables data-driven texture configuration without recompilation.

## External Dependencies
- `OGL_Texture_Def.h` ΓÇö Base texture options and enums (opacity types, blend types, bitmap set constants)
- `InfoTree` ΓÇö Forward declared; used in MML parsing (defined elsewhere)
- Preprocessor: `HAVE_OPENGL` guard (OpenGL support conditional)
