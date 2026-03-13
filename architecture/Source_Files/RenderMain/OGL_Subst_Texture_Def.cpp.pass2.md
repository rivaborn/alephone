# Source_Files/RenderMain/OGL_Subst_Texture_Def.cpp - Enhanced Analysis

## Architectural Role

This module is the **configuration bridge** between MML texture definitions and the GPU-facing OpenGL texture lifecycle. It sits in the RenderMain subsystem's texture management layer, mediating between the parsing/configuration system (XML/InfoTree) and the actual `OGL_TextureOptions` GPU work. During runtime rendering, the visibility and rasterization pipeline queries this module to fetch texture rendering parameters; during level transitions, it orchestrates bulk texture loading/unloading with progress reporting for the UI.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderRasterize / RenderPlaceObjs**: Rendering pipeline calls `OGL_GetTextureOptions(Collection, CLUT, Bitmap)` to retrieve texture configuration during polygon/sprite rasterization
- **XML_MakeRoot.cpp**: MML parser invokes `parse_mml_opengl_texture()` during config load phase to populate texture definitions
- **Level lifecycle management** (inferred): `OGL_LoadTextures()` / `OGL_UnloadTextures()` called during level transitions for GPU memory management
- **OGL_Blitter.cpp**: References in cross-ref index suggest texture loading coordination

### Outgoing (what this file depends on)
- **OGL_TextureOptions** (defined in `OGL_Subst_Texture_Def.h`): Delegates actual GPU work via `Load()` / `Unload()` methods; only stores/retrieves config structures
- **InfoTree parsing** (`XML/InfoTree.h`): All configuration is driven through `read_indexed()`, `read_attr()`, `read_path()` calls; no hardcoded defaults
- **CLUT utilities** (extern): `IsInfravisionTable()`, `IsSilhouetteTable()` for variant detection and fallback logic
- **Progress callback** (extern): `OGL_ProgressCallback(1)` reports per-texture completion for UI feedback during load

## Design Patterns & Rationale

**Cascading Fallback Lookup** (lines 86ΓÇô108): The sophisticated `OGL_GetTextureOptions()` chain reflects real game needsΓÇöexact CLUT match ΓåÆ infravision variant ΓåÆ silhouette variant ΓåÆ all-CLUTs catch-all ΓåÆ hardcoded default. This avoids duplicate config while allowing per-CLUT overrides and variant-specific rendering (thermal/outline vision). Modern engines often flatten this into a single config structure.

**Variant System Translation** (lines 135ΓÇô155): Code actively translates deprecated `CLUT_INFRAVISION_BITMAP_SET` / `CLUT_SILHOUETTE_BITMAP_SET` to the newer `(clut, clut_variant)` pair system, looping through all variants if requested. This is a deprecation path enabling backward compatibility without forcing config updates.

**Collection-Organized Textures**: The `Collections[NUMBER_OF_COLLECTIONS]` array mirrors Marathon's resource collection model, grouping textures by logical sets (walls, sprites, etc.). This grouping simplifies lifecycle management (load/unload entire collection atomically).

**Data-Driven Configuration**: All texture properties (~25 attributes: opacity, blending, bloom, dimensions, images, masks) are parsed from MML; no engine-hardcoded values except the static `DefaultTextureOptions` fallback.

## Data Flow Through This File

1. **Config Load Phase**: MML parser reads `<opengl_texture>` element ΓåÆ `parse_mml_opengl_texture()` ΓåÆ derives `actual_clut` from (clut + variant) pair ΓåÆ inserts `OGL_TextureOptions` into `Collections[coll]` hash map keyed by `(actual_clut, bitmap)`.

2. **Runtime Rendering**: Rasterizer needs texture config ΓåÆ calls `OGL_GetTextureOptions(coll, clut, bitmap)` ΓåÆ cascading lookups return valid pointer (never null) ΓåÆ caller uses config to render.

3. **GPU Lifecycle**: Level load ΓåÆ `OGL_LoadTextures(coll)` iterates hash map, calls `Load()` on each option (GPU memory allocation), reports progress. Level unload ΓåÆ `OGL_UnloadTextures(coll)` iterates, calls `Unload()` (VRAM deallocation).

4. **Config Reset**: `parse_mml_opengl_txtr_clear()` clears collection(s) for reload or total reset.

## Learning Notes

- **Era-specific design**: The CLUT system and collection-based organization reflect Marathon's 1990s architecture. Modern engines use material/shader systems and per-texture metadata.
- **Cascading lookups as a feature**: This file demonstrates intentional "graceful degradation"ΓÇöif no exact match, try infravision, then silhouette, then global fallback. It's a content-authoring convenience: define once for all CLUTs, override for specific ones.
- **Variant system migration**: The code shows how to maintain two configuration paradigms simultaneously (old deprecated CLUT constants + new variant system), useful lesson in API evolution.

## Potential Issues

- **Pointer Invalidation Risk** (line 86): `OGL_GetTextureOptions()` returns `&it->second`, a pointer to a hash map value. If `Collections[coll]` rehashes (e.g., due to insertions during playback), these pointers become dangling. No mutex or versioning protects against this.
- **Thread Safety**: No synchronization around `Collections` global. If `OGL_LoadTextures()` runs in a background thread while rendering calls `OGL_GetTextureOptions()`, race condition is possible on hash map iteration.
- **Static Cleanup**: `DefaultTextureOptions` is never explicitly destructed; relies on implicit cleanup, which may cause issues if `OGL_TextureOptions` manages resources (GPU handles, file paths).
