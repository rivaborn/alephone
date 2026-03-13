# Source_Files/RenderMain/AnimatedTextures.cpp - Enhanced Analysis

## Architectural Role

`AnimatedTextures.cpp` implements the animation subsystem for wall textures within the rendering pipeline, enabling cyclic frame sequences to be played during polygon rasterization. It bridges two critical phases: **configuration-time** setup (parsing XML into frame sequences and timing parameters) and **runtime** application (translating texture descriptors to current animated frames during pixel-level texture lookup). This file is essential for visual effects like flickering lights, flowing lava, and sci-fi interface panels that cycle through bitmaps.

The subsystem is tightly coupled to the rendering backendΓÇöit must update every frame (in `render.cpp`'s main loop) and be consulted for every textured polygon (likely from `RenderRasterize.h/cpp`), making phase-tracking and collection-indexed lookup performance critical.

## Key Cross-References

### Incoming (who depends on this file)

- **RenderMain rendering pipeline** (`render.cpp`, `RenderRasterize*.cpp`): Calls `AnimTxtr_Translate()` during polygon/pixel rendering to map static frame IDs to their current animated substitutes; calls `AnimTxtr_Update()` once per frame to advance animation state
- **XML configuration system** (`XML_MakeRoot.cpp`): Invokes `parse_mml_animated_textures()` at startup/reload to populate animation sequences from MML/XML config
- **Game lifecycle** (`interface.cpp` or equivalent): Calls `reset_mml_animated_textures()` to clear animations before reloading configuration
- Shape descriptor macros and collection frame validation depend on `interface.h` exports (`get_number_of_collection_frames()`)

### Outgoing (what this file depends on)

- **InfoTree** (`InfoTree.h`): XML tree structure and parsing methods (`read_indexed()`, `read_attr()`, `children_named()`) for config deserialization
- **Rendering subsystem macros** (`interface.h`): Shape descriptor extraction/construction macros (`GET_DESCRIPTOR_SHAPE()`, `BUILD_DESCRIPTOR()`) and collection metadata (`get_number_of_collection_frames()`, `NUMBER_OF_COLLECTIONS`)
- **Base types** (`cseries.h`): Integer types, assert/warning macros
- **Standard library**: `<vector>` for dynamic arrays, `<string.h>` for string handling (minimal use)

## Design Patterns & Rationale

| Pattern | Usage | Rationale |
|---------|-------|-----------|
| **Static collection-indexed arrays** | `AnimTxtrList[NUMBER_OF_COLLECTIONS]` | Early-2000s performance optimization: O(1) collection lookup without hash table overhead; assumes `NUMBER_OF_COLLECTIONS` is small and fixed |
| **State machine** | `FramePhase` + `TickPhase` cycling | Encapsulates animation loop logic; deterministic phase advancement enables network sync and replay consistency |
| **In-place modification** | `Translate(short& Frame)` modifies reference argument | Avoids allocation; consistent with functional programming style of descriptor translation |
| **Private state with controlled access** | `AnimTxtr` frame list is private; only `GetFrame()`, `Load()`, `Clear()` exposed | Enforces frame list consistency while allowing selective mutation |
| **Selective application** | `Select` field allows per-texture animation targeting | Enables mods/designers to animate only specific texture IDs within a sequence (e.g., only the "blinking" tile in a grid) |
| **Bidirectional animation via sign** | Negative `NumTicks` = reverse playback | Elegant use of signed integer semantics instead of separate direction boolean |
| **Phase normalization** | `SetTiming()` converts tick overflow into frame advancement | Allows setting animation mid-cycle without manual phase calculation |

## Data Flow Through This File

```
[Initialization & Configuration]
  MML/XML config
       Γåô
  parse_mml_animated_textures() [parses InfoTree]
       Γåô
  AnimTxtr::Load(frame_list) [consumes frame IDs]
  AnimTxtr::SetTiming(ticks, frame_phase, tick_phase) [normalizes phases]
       Γåô
  AnimTxtrList[collection_id].push_back(new_anim)

[Per-Frame Runtime Update]
  AnimTxtr_Update() [called by render.cpp main loop]
       Γåô
  For each collection & animation:
    AnimTxtr::Update() [advances TickPhase, wraps to next frame when tick threshold met]
       Γåô
  Global phase state updated in-place

[Texture Lookup During Rendering]
  shape_descriptor (collection, frame, clut) [from polygon/object]
       Γåô
  AnimTxtr_Translate(descriptor)
       Γåô
  Extract collection_id, frame from descriptor
  Linear search AnimTxtrList[collection_id]
       Γåô
  If animation found:
    AnimTxtr::Translate(frame) [offsets frame by FramePhase, wraps in frame list]
    Returns Frame with updated index
  Γåô
  Validate frame in range [0, num_collection_frames)
  Rebuild descriptor with updated frame
  Return new descriptor to renderer
```

## Learning Notes

1. **Deterministic state advancement**: Unlike modern engines with float-based time, Aleph One uses discrete tick counting (`TickPhase` increments by 1 per frame). This ensures animations sync across networked clients at a fixed 30 FPS tick rateΓÇöessential for deterministic replay and multiplayer consistency.

2. **Collection-specific indexing**: By maintaining separate animation lists per collection, the engine trades memory for O(1) lookupΓÇöa pragmatic choice when `NUMBER_OF_COLLECTIONS` is fixed and small (likely ~10ΓÇô50).

3. **XML over code**: All animation definitions are data-driven, making modding far easier than if hard-coded. By contrast, modern engines often mix procedural and declarative animation.

4. **Phase wrapping complexity**: The `SetTiming()` function reveals that animations can be set at arbitrary mid-cycle points. This is important for save-game resumption or runtime modification but adds complexity to tick/frame normalization.

5. **Selective animation via Select**: The `Select` mechanism is a lightweight alternative to per-sprite animation overrides, enabling precise control without per-instance state overhead.

## Potential Issues

1. **Silent validation failure**: If `Select >= 0` and no frame in `FrameList` matches, `Translate()` returns `false` with no warning. Designers might not realize their animation config is broken.

2. **O(n) linear search per texture**: `AnimTxtr_Translate()` performs a linear search of `AnimTxtrList[collection]` for each textured polygon. Rendering-bound performance could degrade if collections have many animations; however, this likely assumes texture animations are sparse (< 10 per collection).

3. **No phase persistence**: Animation phase state is lost on save/load. Resuming a save-game will restart all animations, creating brief visual discontinuity. Unclear if intentional or oversight.

4. **Unsigned wraparound hazard in SetTiming()**: The normalization logic mixes `int` and `size_t`; if `TickPhase` is set incorrectly large as `size_t`, the cast to `int` can cause underflow. Rare edge case but possible with corrupt config.

5. **No copy constructor/assignment for AnimTxtr**: The class uses auto-generated defaults; if someone later adds non-POD state, silent data corruption could occur. Using `= default;` or `= delete;` would be safer.
