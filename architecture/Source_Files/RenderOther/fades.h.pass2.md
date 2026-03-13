# Source_Files/RenderOther/fades.h - Enhanced Analysis

## Architectural Role

Fades.h implements a **post-processing screen effect layer** that operates orthogonal to the main 3D rendering pipeline. Rather than modifying geometry or texture rendering, it applies color transformations (tinting, gamma correction) globally to the entire rendered frame, with animation over time. This acts as a **composition bridge**: game events (damage, environmental changes, cinematics) trigger state transitions via callbacks (`start_fade()`, `set_fade_effect()`), and the frame-by-frame update loop (`update_fades()`) drives color table interpolation before the final screen composite. The design cleanly separates visual effect concerns from geometry rendering, allowing multiple backend implementations (software rasterizer, OpenGL classic, shader-based) to apply fades uniformly.

## Key Cross-References

### Incoming (who depends on this file)

- **GameWorld/marathon2.cpp**: Calls `start_fade()` on damage events (bullets ΓåÆ red, explosions ΓåÆ yellow), environmental transitions (water/lava entry), and cinematic state changes; calls `update_fades()` each frame in main game loop
- **GameWorld/devices.cpp** (implied): Triggers fades via `start_fade()` on platform activation, teleportation, or scripted events
- **RenderOther/screen_drawing.cpp, computer_interface.cpp**: May call `set_fade_effect()` for immediate tints during UI overlays
- **Lua scripts** (via callbacks): Cinematic scripts control fade via `start_fade()` / `stop_fade()` for choreographed sequences
- **RenderOther/OGL_Faders.h/cpp**: OpenGL-specific implementation of fade rendering; queries current fade state via exported enums

### Outgoing (what this file depends on)

- **CSeries/cscluts.h**: Color table structures (`struct color_table`); reads/writes palette data; calls `gamma_correct_color_table()` 
- **RenderMain/textures.h/cpp**: Likely uses texture lookup tables or palette management
- **XML/InfoTree.h**: MML configuration parsing via `parse_mml_faders()` for declarative fade definitions (e.g., mapping `_fade_red` type to damage feedback callback)
- **RenderOther/screen_drawing.h**: Implied calls to `_set_port_to_*` family to apply modified color tables to screen rendering context

## Design Patterns & Rationale

**1. State Machine + Interpolation**
- Fade types encode pre-choreographed animation sequences (e.g., `_start_cinematic_fade_in` ΓåÆ force black, `_cinematic_fade_in` ΓåÆ interpolate from black over frames)
- Rationale: Decouples visual timing from game logic; cinematics remain smooth regardless of frame rate variations

**2. Callback Function Dispatch via Enums**
- Separate `_tint_fader_type`, `_randomize_fader_type`, `_dodge_fader_type`, etc. enums enable pluggable effect implementations
- Rationale (inferred from comment): XML parser cannot directly invoke C function pointers, so enum values act as indirection for runtime callback selection; supports OpenGL shader-based or software-based filter implementations

**3. Immediate vs. Animated Effects**
- `set_fade_effect(type)` applies tint immediately (e.g., underwater green); `start_fade(type)` animates over frames
- Rationale: Environmental tints are persistent but static; damage feedback and cinematics need smooth transitions

**4. Gamma Correction as Explicit Concern**
- Separate `gamma_correct_color_table()` function with 8 discrete levels (0ΓÇô7)
- Rationale: Pre-shader-era design where color accuracy required manual gamma lookup tables; modern engines use linear color space natively

## Data Flow Through This File

```
Game Event (damage/environment/cinematic)
    Γåô
start_fade(type) / set_fade_effect(type)
    Γåô
[State: fade animation frame counter, source/dest color tables]
    Γåô
update_fades() each render frame
    Γåô
[Interpolate color table between keyframes]
    Γåô
Apply corrected color table to screen context
    Γåô
Screen composite (all pixels remapped through color table)
```

**Key state transitions:**
- Cinematic fades: black ΓåÆ full color ΓåÆ black (multi-stage sequences)
- Damage fades: full color ΓåÆ tinted (red/yellow) ΓåÆ fade out over ~20 frames
- Environmental tints: instant application, persistent until environment changes

## Learning Notes

**What this reveals about Marathon/Aleph One era (mid-1990sΓÇô2000s):**
- **Color-table-based rendering**: The engine renders into indexed 8-bit color, then applies a 256-entry lookup table (`color_table`) to achieve effects like tinting. Modern engines use linear floating-point color and post-process in shader space.
- **Gamma correction as user-facing setting**: 8 gamma levels for monitor calibration (CRT-era concern); modern engines assume sRGB or linear color space by default.
- **Callback function indirection via enums**: Pre-C++ polymorphism design; XML parser specifies effect type as enum, then code dispatches to appropriate filter function. Modern engines would use virtual base classes or function pointers stored in config.
- **MacOS-specific workarounds as code smell**: The `SetFadeEffectDelay()` function indicates a real bug in macOS compositing where dialog boxes would overwrite screen state; the delay allows reapplying the fade after OS painting completes.

## Potential Issues

1. **Hardcoded fade type enumeration**: Adding a new damage type or environmental effect requires modifying the enum and recompiling. A data-driven approach (MML-only definition) would be more extensible.
2. **Global fade state**: Only one fade can be active at a time. Layering effects (e.g., simultaneous damage-red + underwater-green) is not directly supported; would require compositing multiple fades.
3. **MacOS-specific hack in API**: `SetFadeEffectDelay()` is a band-aid for platform-specific bug; suggests similar issues may lurk in other subsystems. The delay value is tuned for a specific OS quirk and may not generalize.
4. **Color table assumption**: Strongly tied to indexed-color rendering. Porting to modern 32-bit RGBA or HDR would require rearchitecting color interpolation and gamma handling.
