# Source_Files/RenderOther/fades.cpp - Enhanced Analysis

## Architectural Role

This file serves as the **visual effect orchestrator** bridging the game world (state changes, events) with display output (screen/OpenGL rendering). It operates at the intersection of three subsystems: GameWorld (which triggers fades), Screen (which manages color tables), and Rendering (which applies effects via OpenGL or software backends). Critically, fades implement a **priority-based effect queue**ΓÇöhigher-priority fades (e.g., damage flashes) interrupt lower-priority ones (e.g., environmental tints), preventing effect "starvation." The file also mediates between **dual rendering backends**: effects either operate on software color tables (via `animate_screen_clut`) or delegate to OpenGL faders when hardware acceleration is active, making it a key compatibility layer.

## Key Cross-References

### Incoming (who depends on this file)
- **GameWorld**: `marathon2.cpp` triggers fades on game events (player damage ΓåÆ `_fade_red`, level transitions ΓåÆ `_cinematic_fade_in`)
- **Interface/Shell**: `interface.cpp` calls `start_fade()` and `set_fade_effect()` for menu and intro sequences
- **Screen system**: `animate_screen_clut()` is called by `recalculate_and_display_color_table()` to push color state to display; `screen.cpp` manages the color table pipeline
- **Rendering**: `OGL_Faders.h` integration allows fades to bypass color-table manipulation entirely when OpenGL is active

### Outgoing (what this file depends on)
- **Screen**: `animate_screen_clut()`, `draw_intro_screen()`, `world_color_table`, `visible_color_table`, `interface_color_table`ΓÇöthe file transforms color tables but does not own them
- **GameWorld**: `dynamic_world->tick_count`, `TICKS_PER_SECOND` for game-time-based fade progression (separate from wall-clock time)
- **OpenGL backend**: `OGL_FaderActive()`, `GetOGL_FaderQueueEntry()`, `OGL_Fader` queueΓÇödelegates effects when GPU rendering is active
- **Movie system**: `Movie::instance()->AddFrame(Movie::FRAME_FADE)` records fade frames for playback/demo mode recording
- **Music system**: `Music::instance()->Idle()` in `full_fade()` allows music to advance during blocking cinematic fades

## Design Patterns & Rationale

**Priority arbitration** (`explicit_start_fade`): New fades only start if no fade is active, their priority ΓëÑ current fade's priority, or they're the same type but the restart throttle (`MINIMUM_FADE_RESTART = 500ms`) has elapsed. This prevents **effect starvation** (low-priority tints blocking high-priority damage) while avoiding **flicker** (same fade restarted every frame). The throttle is architecture-dependent (MacOS odd behavior per code comment).

**Dual-mode time tracking**: Fades interpolate either on **game ticks** (deterministic, used during gameplay for network sync) or **machine ticks** (wall-clock, used for menus/cinematics). This dual-mode design allows fades to be frame-rate-agnostic during gameplay but wall-clock-synchronized during cinematics.

**Effect composition chain**: Environmental effects (water tint) are applied first, then the main fade effectΓÇöboth use the same `fade_proc` signature, allowing arbitrary stacking. The chain reads from one color table and writes to another, enabling ping-pong composition without in-place modifications.

**PRNG isolation**: The `fades_random_seed` static variable uses a simple linear-feedback shift register (LFSR) for reproducible randomization in `_random_transparency_flag`. This avoids contaminating the global RNG but is not cryptographically sound.

## Data Flow Through This File

1. **Input**: Fade trigger event (e.g., `start_fade(_fade_red)`) carries fade type and optional color tables.
2. **Priority check**: `explicit_start_fade()` compares new fade's priority against active fade; throttles same-type restarts.
3. **Initial effect application**: `recalculate_and_display_color_table()` applies both environmental effect and fade effect to copy of original color table.
4. **Per-frame interpolation**: `update_fades()` advances the fade timeline, interpolates transparency linearly from initialΓåÆfinal, applies random noise if flagged, and re-applies effects.
5. **Dual output**: Processed color table goes either to `animate_screen_clut()` (software path) or to OpenGL fader queue (hardware path).
6. **Deactivation**: When `phase >= period`, fade completes and `FADE_IS_ACTIVE` flag clears; final transparency is applied once more before returning control.

## Learning Notes

- **Fixed-point transparency**: Uses `_fixed` (16.16 fixed-point) for opacity calculations, enabling sub-pixel precision without floating-point overheadΓÇöidiomatic for 1990s engine code.
- **Singleton global state**: The `fade` struct is a single global instance (not an array), reflecting assumption of only one fade active at a time. Priority arbitration prevents concurrent fades rather than supporting them.
- **Color table copies**: Environmental and fade effects operate on copies of `original_color_table` to `animated_color_table`; original is never modified, allowing clean rollback or re-application.
- **MML customization architecture**: `parse_mml_faders()` / `reset_mml_faders()` maintain backup arrays for defaults, allowing runtime reset without re-parsingΓÇöuseful for development and mod iteration without restart.
- **Gamma as display calibration**: Gamma correction is applied *after* fade effects, reflecting the assumption that gamma is a display property (brightness curve), not a creative effect layer.

## Potential Issues

- **MINIMUM_FADE_RESTART throttle is fragile**: Hardcoded to 500ms with no user configurability; if game ticks stop temporarily (e.g., pause menu), the throttle breaks and same fade can re-trigger unexpectedly.
- **FadeEffectDelay workaround**: The delay counter (MacOS oddities per comments) introduces asymmetry between effect and fade timing; future platforms or refactors may leave this stale.
- **Effect starvation via priority**: Lower-priority environmental tints can be completely suppressed by higher-priority damage flashes for extended periods; no timeout prevents permanent suppression.
- **Random transparency LFSR**: Simple XOR LFSR has short period (~64K) and poor statistical properties; high-frequency flicker effects may show visible patterns.
