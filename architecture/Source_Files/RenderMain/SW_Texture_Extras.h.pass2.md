# Source_Files/RenderMain/SW_Texture_Extras.h - Enhanced Analysis

## Architectural Role

This file is the **software rasterizer's texture metadata and opacity-table manager**, sitting between the shape collection system and the `scottish_textures` rasterizer. It caches pre-computed opacity lookup tables per texture, enabling fast per-pixel opacity blending during software rasterizationΓÇöa pre-GPU optimization era pattern. The singleton ensures one global registry of texture properties per engine instance, with per-collection partitioning reflecting Marathon's WAD-based resource structure (32 collections max).

## Key Cross-References

### Incoming (who depends on this file)
- **`scottish_textures.cpp`** ΓÇö Calls `GetTexture()` to retrieve opacity tables (`opac_table()`) during polygon rasterization; reads `opac_type()` to select blending mode
- **XML/MML subsystem** ΓÇö `parse_mml_software()` is called during engine initialization to populate texture properties from MML config (likely from `XML_MakeRoot.cpp` during level load)
- **Rendering pipeline initialization** ΓÇö `Load(collection)` called when a shape collection is loaded; `Unload()` and `Reset()` during cleanup

### Outgoing (what this file depends on)
- **`shape_descriptors.h`** ΓÇö Imports `shape_descriptor` typedef and `NUMBER_OF_COLLECTIONS` macro; encodes collection/shape indices
- **`shapes.cpp`** ΓÇö Indirectly: texture properties are looked up from shape collection data during `build_opac_table()`
- **InfoTree (XML subsystem)** ΓÇö `parse_mml_software(const InfoTree&)` decodes texture config; forward-declared only in this header

## Design Patterns & Rationale

1. **Singleton (Meyers)**: `instance()` uses static pointer initialized on first access. No thread locks (pre-SMP era assumption). Instance never freed (acceptable for engine lifetime).

2. **Lazy Table Building**: `build_opac_table()` not called in constructor; caller must invoke explicitly. Trades memory for initialization speedΓÇötables are expensive and only built when needed.

3. **Per-Collection Partitioning**: `texture_list[NUMBER_OF_COLLECTIONS]` mirrors WAD resource structure. Enables `Load(Collection)` / `Unload(Collection)` granularity, avoiding recomputation when unused collections are unloaded.

4. **Lookup Table Optimization**: Opacity parameters (scale, shift, type) pre-compute a `uint8` lookup table. This was critical for software rendering to avoid per-pixel floating-point ops; modern GPUs do this in shaders.

## Data Flow Through This File

```
Shape descriptor (collection/shape indices)
    Γåô
MML config (opac_type, opac_scale, opac_shift) ΓÇö via parse_mml_software()
    Γåô
SW_Texture object stores metadata
    Γåô
build_opac_table() ΓÇö queries shape pixel data, applies scale/shift, generates [0..255]ΓåÆopacity LUT
    Γåô
opac_table() pointer returned to rasterizer
    Γåô
Rasterizer indexes lookup table: opac_value = table[pixel_color]
```

## Learning Notes

- **Era-specific optimization**: Pre-computed lookup tables were essential for software rendering. Modern engines defer this to fragment shaders, trading memory for parallelism.
- **MML extensibility**: The texture system was designed to accept user-defined opacity parameters via MML, not hardcoded in C++. This reflects Aleph One's modding-friendly design.
- **Collection-aware design**: The per-collection vectors reflect Marathon's WAD-based resource loadingΓÇötextures are grouped by collection for efficient bulk load/unload.
- **Stateful singleton**: Unlike functional initialization, this singleton accumulates state across the engine lifetime, simplifying global access but complicating testing and teardown.

## Potential Issues

- **No thread safety**: Static singleton not guarded by mutex; concurrent calls to `instance()` or MML parsing could race (acceptable for deterministic game loop; problematic if multi-threaded).
- **Null pointer risk**: `opac_table()` returns nullptr if table not built; callers must null-check or crash on dereference.
- **Silent failures**: `build_opac_table()` implementation not visible; no error indication if shape lookup fails.
- **Memory leak**: Static instance allocated once, never freed. Acceptable for singleton lifetime == program lifetime.
- **Descriptor validation**: `descriptor()` setter accepts any `shape_descriptor` value; invalid shape indices cause undefined behavior in `build_opac_table()`.
