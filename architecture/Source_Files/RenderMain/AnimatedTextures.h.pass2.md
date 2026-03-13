# Source_Files/RenderMain/AnimatedTextures.h - Enhanced Analysis

## Architectural Role

This header bridges **texture animation state management** with the **rendering pipeline** via a minimal public interface. It decouples animation frame sequencing (updated once per 30 FPS game tick) from texture lookup (called per polygon during rasterization). The design allows the rendering backend to request current animated texture variants without knowledge of animation timing, frame selection, or configuration detailsΓÇöall encapsulated privately in the implementation file.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderRasterize / RenderPlaceObjs**: Calls `AnimTxtr_Translate()` during polygon/sprite rendering to map base texture descriptors ΓåÆ animated frame variants
- **RenderMain render loop**: Calls `AnimTxtr_Update()` once per frame to advance animation timers after world state updates
- **XML configuration system** (via `parse_mml_animated_textures`): Marathon Markup Language loader for level/scenario setup
- **Preferences/reset flow** (via `reset_mml_animated_textures`): Clears animation tables during shutdown or config reload

### Outgoing (what this file depends on)
- **shape_descriptors.h**: Defines the `shape_descriptor` type (16-bit packed field: collection + shape + CLUT indices) used as texture identifiers
- **XML/InfoTree subsystem**: `InfoTree` class for structured configuration parsing; must be forward-declared here, fully defined elsewhere
- Implementation file (AnimatedTextures.cpp): Holds actual animation state tables, frame lookup tables, and timing logic

## Design Patterns & Rationale

**Opaque State Encapsulation**: The `.h` file exposes only four functions; all state (animation timers, frame tables, configuration) is private to the `.cpp` file. This is a classic **data-hiding** pattern that:
- Prevents accidental direct state mutation from renderers
- Allows safe reordering of animation updates relative to rendering without exposing frame-counting details
- Decouples animation frame selection algorithm from clients

**Stateless Translation Function**: `AnimTxtr_Translate()` is effectively a **pure lookup function** (no visible side effects) that maps shape_descriptor ΓåÆ shape_descriptor. This allows it to be called from hot rendering paths (per-polygon in `RenderRasterize`) without synchronization concerns or re-entrancy risks.

**Deferred Configuration**: Parse and reset functions separate policy (when to load/clear) from mechanism (what animation tables exist). This enables runtime reconfiguration without re-initializing the entire renderer.

## Data Flow Through This File

```
Configuration Phase:
  MML/XML file ΓåÆ parse_mml_animated_textures() ΓåÆ [internal: build frame tables]
                                                Γåô
                                    (stored in static/global state)
                                    
Per-Frame Rendering Loop:
  GameWorld.Update() (30 FPS tick) ΓåÆ AnimTxtr_Update() 
                                      Γåô (advance animation counters)
                                      
  Rendering Phase:
    RenderRasterize iterates polygons
      for each polygon ΓåÆ shape_descriptor ΓåÆ AnimTxtr_Translate()
                            Γåô (pure lookup: base ΓåÆ current animated frame)
                        translated_descriptor (sent to rasterizer)
```

**Key insight**: The translation is **deferred** until rasterization time. The animation state (current frame index) is global/static and updated once per tick; lookup is stateless and read-only. This separation allows the renderer to be frame-rate-independent (can render >30 FPS with interpolated frames, all viewing the same animation state).

## Learning Notes

This design exemplifies **decoupling of state management from consumption**:

- **Marathon/Aleph One era pattern**: Stateful animation update separated from stateless translation lookup, common in engines before ECS/scripting made per-entity animation straightforward
- **Modern parallel**: Would likely use a component-based system where each animated texture entity tracks its own frame timer, or use a shader-based frame selection (texture atlasing with frame index as uniform)
- **Texture descriptor packing**: The 16-bit `shape_descriptor` encoding (collection.5 | shape.8 | clut.3) is space-efficient for 1990s memory constraints but opaque to readers; see `shape_descriptors.h` for the bit manipulation macros

**Idiomatic to this engine**: The MML (Marathon Markup Language) configuration system relies heavily on `InfoTree` parse trees, a pattern repeated across Aleph One for all major subsystems (weapons, items, monsters, rendering effects, Lua bindings). This centralizes customization without recompilation.

## Potential Issues

1. **No thread safety guarantees**: If `AnimTxtr_Update()` is called from one thread and `AnimTxtr_Translate()` from another (e.g., background rendering thread), a race condition on global animation state could cause frame tearing or descriptor lookups returning stale/invalid indices. Not likely in single-threaded 30 FPS engines, but risky if parallelized.

2. **Silent passthrough for non-animated textures**: `AnimTxtr_Translate()` returns the input unchanged if the texture is not animated. This is safe but provides no indication whether translation occurredΓÇömakes it hard to debug if an expected animated texture fails to animate (typo in config, texture ID mismatch).

3. **Circular animation dependency risk**: If `parse_mml_animated_textures()` is called during level load and references undefined collection/shape indices (e.g., custom shapes that failed to load), the implementation likely bounds-checks silently, losing the animation silently. Error reporting mechanism unclear from header alone.
