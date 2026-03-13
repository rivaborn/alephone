# Source_Files/Input/mouse.h - Enhanced Analysis

## Architectural Role

Mouse input is a **bridge layer** connecting SDL2 hardware events to the engine's unified scancode-based input system. Rather than passing raw input data through the game loop, mouse state is accumulated per-frame and then projected into the existing keymap format, allowing the core game logic (which evolved from keyboard-driven input) to consume mouse actions without modification. This design reflects an input architecture that evolved from discrete keypresses and intentionally preserves that abstraction even as it adds continuous input (mouselook, mouse movement) and mouse button emulation.

## Key Cross-References

### Incoming (who depends on this file)
- **Input subsystem** (`joystick_sdl.cpp`, Shell/Interface layer): calls `enter_mouse()`, `exit_mouse()`, `mouse_idle()`, and `pull_mouselook_delta()` during frame loop
- **Event handlers** (SDL event dispatch): invoke `mouse_moved()` and `mouse_scroll()` from SDL event callbacks
- **Player aim system** (GameWorld physics): reads mouselook deltas via `pull_mouselook_delta()` to update view angles each frame
- **Keymap consumers** (game action logic): read synthesized mouse-button scancodes (400ΓÇô406) from the shared keymap after `mouse_buttons_become_keypresses()` fills it
- **Preferences/Settings**: affect mouse sensitivity/acceleration applied in `mouse_moved()`

### Outgoing (what this file depends on)
- **world.h**: provides `fixed_yaw_pitch` struct for rotation representation
- **SDL2**: underlying mouse device state, relative motion tracking, cursor positioning (via implementation)
- **Keymap buffer**: shared Uint8 array (likely 512 scancodes) written to by `mouse_buttons_become_keypresses()`
- **Preferences subsystem**: reads stored sensitivity/acceleration settings (via implementation)

## Design Patterns & Rationale

**Scancode Aliasing for Mouse Buttons**
- Mouse buttons 1ΓÇô7 are mapped to pseudo-scancodes (400ΓÇô406), repurposing the same 512-entry keymap that keyboard uses
- Rationale: Game logic (`process_action_flags()`, terminal input, etc.) is built around discrete key events; aliasing buttons avoids duplicating this logic
- Tradeoff: Loses ability to distinguish button press timing or detect button chords; encoded as a "semi-hacky scheme" in the source comment

**Polling-Based Frame Loop**
- `mouse_idle(short type)` called every frame; `pull_mouselook_delta()` extracts accumulated motion, resetting the accumulator
- Rationale: Game runs at fixed 30 FPS tick rate; polling ensures deterministic, replayed-able input within the simulation timestep
- Tradeoff: Wastes mousemove events between frames if polling is infrequent; modern engines use event-driven queues

**Relative Mouse Mode for Mouselook**
- `recenter_mouse()` resets cursor to screen center; subsequent `mouse_moved(delta_x, delta_y)` events are relative deltas
- Rationale: FPS-style mouselook requires mouse trapping to prevent cursor hitting screen edge
- Idiomatic for this era: SDL relative mouse mode was the standard approach before raw input APIs

**Type-Parameterized Input Modes**
- `enter_mouse(short type)` / `exit_mouse(short type)` suggest different behavioral modes (e.g., gameplay vs. menu)
- Rationale: Allows decoupling of mouse behavior from gameplay state; menus might not recenter cursor or apply sensitivity
- Reflects legacy design where input subsystems are mode-aware rather than passively polling

**Fixed-Point Rotation Deltas**
- `fixed_yaw_pitch` uses fixed-point precision rather than floats
- Rationale: Ensures deterministic network play (no floating-point rounding divergence); traces back to Marathon's LAN multiplayer era
- Modern engines: use floats with canonical floating-point implementations or integer-based network reconciliation

## Data Flow Through This File

**Per-Frame Mouselook Loop:**
1. **Event Phase**: SDL mouse-move events ΓåÆ `mouse_moved(delta_x, delta_y)` accumulates deltas into internal state
2. **Idle Phase**: `mouse_idle(short type)` called once per frame (allows sensitivity/acceleration curve application)
3. **Query Phase**: `pull_mouselook_delta()` returns accumulated yaw/pitch, resets accumulator to (0, 0)
4. **Application Phase**: Player physics reads returned deltas, updates view angles

**Per-Frame Button/Scroll Loop:**
1. **Event Phase**: SDL button-down/button-up and scroll events ΓåÆ internal button state updated
2. **Keymap Phase**: `mouse_buttons_become_keypresses(ioKeyMap)` writes button states (1ΓÇô7) as scancodes (400ΓÇô406) into keymap
3. **Game Logic**: Standard key processing reads keymap, triggers actions (e.g., fire weapon, scroll inventory)

**Lifecycle:**
- `enter_mouse(type)`: Allocate state, potentially enable relative mouse mode
- Repeated per-frame: `mouse_idle()`, `pull_mouselook_delta()`, `mouse_buttons_become_keypresses()`
- `recenter_mouse()`: Called opportunistically (e.g., after alt-tab) to reset cursor
- `exit_mouse(type)`: Release state, disable relative mode

## Learning Notes

**What a Developer Learns:**
1. **Unified Input Abstraction**: Keyboard, mouse, and gamepad all feed into a single 512-entry scancode keymap; this is a viable pattern for games that value input simplicity over responsiveness
2. **Frame-Driven vs. Event-Driven**: This engine polls input per simulation frame rather than queueing events; requires `mouse_idle()` to be called religiously
3. **Fixed-Point for Determinism**: Network play shaped the use of `fixed_yaw_pitch` rather than floats; many modern networking implementations avoid this complexity

**What's Idiomatic to Aleph One / 1990sΓÇô2000s Era:**
- ButtonΓåÆkeymap projection (mouse buttons 1ΓÇô7 as scancodes 400ΓÇô406) is a pragmatic hack; modern engines pass `MouseButton` enums directly
- No support for mouse acceleration curves beyond frame-based clamping; modern input APIs separate raw delta from processed aim input
- Type-parameterized input modes (`enter_mouse(type)`) assumes discrete UI/gameplay states; modern engines have mode-agnostic input systems
- Relative mouse positioning via `recenter_mouse()` and SDL relative-mode; newer approaches use raw input APIs (Windows RawInput, Linux /dev/input)

**Missing from This Header:**
- No explicit mouse acceleration/sensitivity parameters (inferred to be in implementation or Preferences)
- No mouse lock/capture API (implied to be in `enter_mouse`)
- No scroll direction preference (boolean `up` suggests hard-coded behavior)

## Potential Issues

1. **Type Parameter Ambiguity**: `enter_mouse(short type)` and `exit_mouse(short type)` don't document what `type` values are valid; caller must know these constants or risk silent failure
2. **Frame-Dropped Input**: If `mouse_idle()` or `pull_mouselook_delta()` is skipped even once, frame drops mousemove events; polling-based systems are brittle if the caller forgets a call
3. **Scancode Collision Risk**: Virtual scancodes 400ΓÇô406 for mouse buttons must not collide with any valid keyboard scancode; if a future port uses extended keycodes, this could break
4. **No Mouse State Query API**: Unlike keyboard keymaps (which can be sampled directly), mouse state is opaque; debugging input issues requires adding logging to `mouse_moved()` rather than inspecting current state
5. **Hardcoded Scroll Wheel Behavior**: `mouse_scroll(bool up)` doesn't map to configurable actions; scroll wheel behavior is fixed rather than configurable like keyboard key bindings

---

**Cross-Cutting Insight**: This file exemplifies the **Legacy Abstraction Layer** patternΓÇöa wrapper that preserves an older input model (keymap) while incrementally adding new input types (mouselook, scroll). The design works but creates seams: mouselook bypasses the keymap, buttons enter it, scroll is half-way between. A greenfield engine would unify these into a single input event stream or state buffer.
