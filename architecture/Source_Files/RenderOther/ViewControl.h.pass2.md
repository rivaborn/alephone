# Source_Files/RenderOther/ViewControl.h - Enhanced Analysis

## Architectural Role

ViewControl.h is a **configuration facade** that centralizes camera and environment rendering parameters. It bridges the XML/preferences system to the rendering pipeline and game logic, allowing per-level and per-texture customization of visual appearance without exposing implementation details. Critical to supporting multiple Marathon engine versions (1, 2, Infinity) which have different rendering conventions.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderMain subsystem** calls `View_FOV_*()` accessors during camera setup (`render.cpp`, `RenderVisTree.cpp`) to configure projection
- **Main game loop** queries `View_DoFoldEffect()`, `View_DoStaticEffect()`, and interlevel teleport flags to apply visual effects during level transitions
- **HUD/UI rendering** retrieves `GetOnScreenFont()` for on-screen text (menus, messages, HUD elements) via `screen_drawing.cpp`
- **Overhead map system** checks `View_MapActive()` to determine if map display is available
- **XML initialization** in `XML_MakeRoot.cpp` calls `parse_mml_view()` and `parse_mml_landscapes()` during startup

### Outgoing (what this file depends on)
- **RenderOther subsystem**: includes `FontHandler.h` (provides `FontSpecifier`), `shape_descriptors.h` (provides `shape_descriptor` type)
- **GameWorld**: includes `world.h` (provides `angle` type for landscape azimuth parameter)
- **XML system**: depends on forward-declared `InfoTree` class for MML parsing (defined in `XML/InfoTree.h`)

## Design Patterns & Rationale

**Accessor Pattern** ΓÇö All functions are read-only getters (except `View_AdjustFOV` which modifies caller's parameter, not internal state). This encapsulation allows:
- Implementation to move between static variables, configuration files, or database without changing the API
- Version-specific defaults to be swapped at runtime (e.g., Marathon 1 vs. 2 landscape repeat mode)

**Per-Texture Configuration** ΓÇö `LandscapeOptions` is keyed by `shape_descriptor`, enabling:
- Fine-grained customization per environment (different outdoor sky vs. indoor texture scaling)
- Marathon format compatibility workarounds (e.g., `VertRepeat` toggles between M1 tiling and M2 clamping)
- OpenGL-specific compensation (`OGL_AspRatExp`) for power-of-2 texture aspect ratio constraints

**State Reset Pattern** ΓÇö Paired `parse_mml_*()` and `reset_mml_*()` functions enable:
- Clean reload of configuration without full engine restart
- Mod testing without recompiling
- Multiplayer consistency checks (verify all players loaded same settings)

## Data Flow Through This File

1. **Initialization**: `XML_MakeRoot` ΓåÆ `parse_mml_view()` + `parse_mml_landscapes()` populate internal configuration state from MML files
2. **Per-Frame Rendering**: Rendering pipeline queries `View_FOV_*()` ΓåÆ camera projection; calls `View_GetLandscapeOptions(shape)` ΓåÆ texture-specific scaling applied when drawing environment
3. **FOV Transitions**: Game logic calls `View_AdjustFOV(current, target)` each frame ΓåÆ smooth FOV interpolation for vision mode changes (extravision, tunnel vision)
4. **Teleport Effects**: Game loop checks teleport flags ΓåÆ conditionally executes fold/static visual effects during player teleport or level transition
5. **HUD Rendering**: UI system calls `GetOnScreenFont()` ΓåÆ renders all on-screen text with consistent typography

## Learning Notes

**Idiomatic to Aleph One era** (early 2000s):
- Heavy reliance on global/static state (no OOP composition for configuration)
- XML-based modding system predates modern config formats (TOML, YAML)
- Landscape azimuth uses legacy `angle` type (512 units = full circle) rather than radians or degrees
- Separator of concerns via thin header boundaries (implementation details hidden from callers)

**Modern engines** would:
- Use object-oriented camera/environment classes with composition
- JSON or YAML for configuration
- Runtime reflection instead of manual parse/reset pairs
- Per-material properties rather than per-shape lookup tables

**Engine-Specific Knowledge**:
- The `OGL_AspRatExp` field reveals a workaround: Bungie's original Marathon landscape textures don't have power-of-2 heights, so this exponent compensates when rendering in OpenGL (which prefers POT dimensions)
- `VertRepeat` boolean shows engine compatibility philosophy: support multiple source formats transparently

## Potential Issues

1. **Unsafe pointer return**: `View_GetLandscapeOptions()` returns naked `LandscapeOptions*`; if shape has no custom options, returns null without convention documentation. Callers must null-check or crash.

2. **Mutable FOV reference parameter**: `View_AdjustFOV(float& FOV, ...)` modifies caller's variable; if called with const/temporary, will not compile (safe), but mutable reference semantics are unconventional for a "view" accessor.

3. **No synchronization**: If `parse_mml_landscapes()` is called while rendering thread is querying `View_GetLandscapeOptions()`, potential data race on landscape lookup table. No mutex protection evident.

4. **Missing documentation**: Units for FOV values (degrees? radians?) not specified in header; callers must infer from usage or implementation.
