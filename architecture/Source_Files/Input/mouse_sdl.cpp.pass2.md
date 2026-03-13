# Source_Files/Input/mouse_sdl.cpp - Enhanced Analysis

## Architectural Role

This file implements the SDL backend for mouse input abstraction, translating low-level SDL events into the engine's unified input framework. It serves as a critical bridge between raw hardware input (relative motion, button state, scroll events) and the game world's player control system, supporting both camera control (mouselook with configurable acceleration curves) and button-as-keypress emulation. The file encapsulates platform-specific SDL mouse handling so that higher-level Input and GameWorld subsystems remain device-agnostic.

## Key Cross-References

### Incoming (who depends on this file)

- **Input subsystem** (`Source_Files/Input/joystick_sdl.cpp`, `Source_Files/Input/mouse.h`): This file is the SDL implementation of the mouse interface declared in `mouse.h`; functions are called by input polling code
- **GameWorld** (`Source_Files/GameWorld/player.cpp`, `Source_Files/GameWorld/physics.cpp`): Calls `pull_mouselook_delta()` each frame to retrieve yaw/pitch deltas for camera rotation (implicit via unified action queue)
- **Shell/Screen** (`Source_Files/RenderOther/screen.h`): Calls `enter_mouse()`, `exit_mouse()`, `recenter_mouse()`, `hide_cursor()`, `show_cursor()` during game state transitions and window resize
- **SDL event pump**: Calls `mouse_moved()` and `mouse_scroll()` from event handlers to accumulate input
- **Input keymap population**: `mouse_buttons_become_keypresses()` is called by input polling to convert button state into the unified keyscan array

### Outgoing (what this file depends on)

- **SDL2**: `SDL_SetRelativeMouseMode()`, `SDL_SetHint()`, `SDL_GetMouseState()`, `SDL_ShowCursor()`
- **Preferences** (`input_preferences` global from `preferences.h`): Reads `sens_horizontal`, `sens_vertical`, `mouse_accel_type`, `mouse_accel_scale`, `classic_vertical_aim`, `modifiers` bitmask
- **Screen** (`MainScreenCenterMouse()` from `screen.h`): Repositions OS cursor to center when entering mouselook or on screen resize
- **Engine types** (`world.h`): `fixed_angle`, `fixed_yaw_pitch`, `_fixed` for geometry and fixed-point math

## Design Patterns & Rationale

**Snapshot Accumulation Pattern**: Raw SDL input events are accumulated into frame-local snapshots (`snapshot_delta_x/y`, `snapshot_delta_scrollwheel`) which persist across event pump calls. This allows multiple input events to be collected into a single meaningful delta per frame, preventing frame-skipping issues and allowing the game loop to process input at fixed 30 FPS regardless of event frequency.

**Unified Input Abstraction**: Mouse buttons and scroll wheel are converted to pseudo-keyscan codes (400ΓÇô406 in `AO_SCANCODE_BASE_MOUSE_BUTTON` range), allowing the rest of the engine to treat mouse input identically to keyboard input. This unifies three input devices (keyboard, mouse, gamepad) through a single scancode-based interface, reducing coupling throughout GameWorld and shell code.

**Sensitivity/Acceleration Composition**: The calculation chain `(preference ├ù base_scale ├ù magnitude_factor)` is applied during `mouse_idle()`, decoupling raw input from game feel. The magic constant `128/66` tuning assumes `_mouse_accel_none` and can be overridden by user preferences at runtime, allowing post-release tuning without code changes.

**Button Masking ("Press-to-Enable")**: The `button_mask` uses `button_mask |= ~orig_buttons` to require button release before re-enabling. This prevents accidental shots fired if a user holds a mouse button from GUI interaction before entering gameplay (a UX issue on hybrid UI/game transitions). Buttons remain masked until explicitly released, then re-enabled.

## Data Flow Through This File

```
SDL Event Stream
  Γåô [mouse_moved(), mouse_scroll()]
Snapshot Accumulation (snapshot_delta_x/y, snapshot_delta_scrollwheel)
  Γåô [mouse_idle() per frame]
Raw Delta + Preference Application
  Γö£ΓöÇ Inversion (if _inputmod_invert_mouse)
  Γö£ΓöÇ Sensitivity (horizontal/vertical factors from preferences)
  Γö£ΓöÇ Classic vertical aim reduction (├╖4 if enabled)
  ΓööΓöÇ Acceleration curve (classic: MIX toward 1/32 or 1/8 factor)
  Γåô
mouselook_delta (fixed_yaw_pitch)
  Γåô [pull_mouselook_delta() called by game loop]
GameWorld Player Camera Rotation

Button/Scroll State
  Γåô [mouse_buttons_become_keypresses()]
Keymap Array Population (indices 400ΓÇô406)
  Γåô
Unified Input System ΓåÆ Player Action Flags
```

## Learning Notes

- **Era-specific design**: This code reflects early-2000s game input architecture (pre-DirectInput abstractions, pre-Steam input APIs). The snapshot pattern and pseudo-keymap conversion were standard to avoid per-device logic in game logic.
- **Magic number archaeology**: The `128/66` constant suggests empirical tuning for Marathon's perceived camera sensitivity. Modern engines use lookup tables or spline curves instead.
- **Classic mode concessions**: The `classic_vertical_aim` flag reducing Y sensitivity by 4├ù indicates the original Marathon had non-1:1 mouse axis ratios for vertical aimΓÇöa deliberate game design decision preserved for backward compatibility.
- **Relative mouse normalization**: SDL's relative mouse mode (with optional warp) abstracts platform differences (macOS center-screen vs. window-edge behavior), centralizing OS-specific quirks here.
- **Fixed-point arithmetic**: `fixed_angle` and `fixed_yaw_pitch` use integer arithmetic for determinism in networked/replayed games, avoiding floating-point rounding variability.

## Potential Issues

- **Unvalidated preference pointer**: `input_preferences` global is dereferenced without null checks. If preferences fail to initialize, this crashes ungracefully. Consider defensive null-check in `mouse_idle()`.
- **Magic constant opacity**: `128/66` lacks a comment explaining its derivation (assumed sensitivity scaling). Future maintainers may incorrectly adjust it.
- **Scroll accumulation loss**: Multiple rapid scroll events in one frame may coalesce into a single `┬▒1` delta in `snapshot_delta_scrollwheel`, losing granularity. Unlikely with typical hardware, but document the limitation.
- **Button mask state non-obvious**: The `button_mask |= ~orig_buttons` logic implements "press-to-enable" implicitly. Add an inline comment: `// Button must be released (not in orig_buttons) before it can be enabled`.
