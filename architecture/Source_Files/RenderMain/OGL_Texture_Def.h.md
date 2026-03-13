# Source_Files/RenderMain/OGL_Texture_Def.h

## File Purpose
Defines OpenGL texture configuration structures and constants for wall/sprite texture substitutions and model skins in the Aleph One engine. Handles multiple CLUT (color lookup table) variants, opacity modes, blend types, and special effects like infravision/silhouette rendering.

## Core Responsibilities
- Define bitmap set enumeration for color tables, infravision, and silhouette variants
- Enumerate CLUT variants (normal, infravision, silhouette)
- Define opacity types (Crisp, Flat, Avg, Max) and alpha blending modes
- Provide helper functions to identify special CLUT types
- Define `OGL_TextureOptionsBase` struct for texture configuration with opacity, blending, and bloom controls
- Manage image loading with FileSpecifier paths and ImageDescriptor objects

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OGL_TextureOptionsBase` | struct | Base configuration for textures: opacity/blend modes, image files, bloom settings, premultiplication flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `INFRAVISION_BITMAP_SET` | enum constant | global | Bitmap set index for infravision color table variant |
| `SILHOUETTE_BITMAP_SET` | enum constant | global | Bitmap set index for silhouette color table variant |
| `NUMBER_OF_OPENGL_BITMAP_SETS` | enum constant | global | Total count: `3 * MAXIMUM_CLUTS_PER_COLLECTION + 2` |
| `ALL_CLUTS` | int const | global | Sentinel value (-1) meaning "all color tables" |
| Opacity type enums | enum constants | global | `OGL_OpacType_Crisp`, `OGL_OpacType_Flat`, `OGL_OpacType_Avg`, `OGL_OpacType_Max` |
| Blend type enums | enum constants | global | `OGL_BlendType_Crossfade`, `OGL_BlendType_Add`, premultiplied variants |

## Key Functions / Methods

### IsInfravisionTable
- Signature: `static inline bool IsInfravisionTable(short CLUT)`
- Purpose: Check if a CLUT index refers to an infravision color table
- Inputs: `CLUT` ΓÇö color table index
- Outputs/Return: `true` if CLUT is infravision variant
- Side effects: None
- Calls: None
- Notes: Checks both the global infravision set and per-collection variants

### IsSilhouetteTable
- Signature: `static inline bool IsSilhouetteTable(short CLUT)`
- Purpose: Check if a CLUT index refers to a silhouette color table
- Inputs: `CLUT` ΓÇö color table index
- Outputs/Return: `true` if CLUT is silhouette variant
- Side effects: None
- Calls: None
- Notes: Checks both the global silhouette set and per-collection variants

### OGL_TextureOptionsBase (constructor)
- Signature: `OGL_TextureOptionsBase()`
- Purpose: Initialize texture configuration with sensible defaults
- Inputs: None
- Outputs/Return: Initializes member fields
- Side effects: Sets default opacity type (Crisp), scales (1.0), blends (Crossfade), bloom values, premultiplication flags
- Calls: None
- Notes: Initializer list sets ~16 member fields; OpacityShift defaults to 0, MinGlowIntensity to 1

### Load / Unload
- Signature: `void Load()`, `void Unload()`
- Purpose: Load/unload texture image data from disk
- Inputs/Outputs: Not inferable from this header
- Side effects: File I/O, memory allocation
- Calls: Likely calls `ImageDescriptor::LoadFromFile()` and related image loaders
- Notes: Implemented elsewhere; virtual method `GetMaxSize()` suggests subclasses may override

## Control Flow Notes
This header defines configuration and constants used during texture initialization and rendering setup. It is **not** part of frame/render loops; rather, it structures texture metadata that is parsed from configuration files (likely MML) and used to configure OpenGL rendering state during asset load time.

## External Dependencies
- `shape_descriptors.h` ΓÇö provides `MAXIMUM_CLUTS_PER_COLLECTION` macro
- `ImageLoader.h` ΓÇö provides `ImageDescriptor` class and `FileSpecifier` type
- `<vector>` ΓÇö STL vector (included but not directly used in this header)
- `HAVE_OPENGL` ΓÇö preprocessor guard
