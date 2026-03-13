# Source_Files/Input/mouse_sdl.cpp

## File Purpose
Implements SDL-specific mouse input handling for in-game mouselook, including sensitivity/acceleration control, mouse button emulation as keypresses, scroll wheel input, and cursor management.

## Core Responsibilities
- Initialize and shutdown in-game mouse input mode with relative mouse tracking
- Calculate mouselook yaw/pitch deltas from raw mouse movements with configurable sensitivity, inversion, and acceleration
- Convert SDL mouse buttons to pseudo-keyscan codes for the input system
- Map scroll wheel movements to button-like input
- Manage cursor visibility and recenter on screen size changes

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| mouse_active | bool | static | Flags whether in-game mouselook mode is enabled |
| button_mask | uint8 | static | Bitmask of enabled mouse buttons; buttons disabled until released (prevents accidental firing when entering game) |
| mouselook_delta | fixed_yaw_pitch | static | Accumulated yaw and pitch angle deltas for this frame |
| snapshot_delta_scrollwheel | _fixed | static | Current frame's scroll wheel accumulation (+1, 0, or ΓêÆ1) |
| snapshot_delta_x, snapshot_delta_y | int | static | Current frame's raw mouse movement deltas in pixels |

## Key Functions / Methods

### enter_mouse
- Signature: `void enter_mouse(short type)`
- Purpose: Activate in-game mouse input with relative motion tracking
- Inputs: `type` ΓÇô input device type; only activates if type != _keyboard_or_game_pad
- Outputs/Return: (none)
- Side effects: Centers mouse, enables SDL relative mode, resets all deltas, clears button mask, sets mouse_active=true
- Calls: MainScreenCenterMouse(), SDL_SetHint(), SDL_SetRelativeMouseMode()
- Notes: Button mask is cleared to prevent a shot firing if user enters game with mouse button held from GUI interaction

### exit_mouse
- Signature: `void exit_mouse(short type)`
- Purpose: Deactivate in-game mouse input
- Inputs: `type` ΓÇô input device type; only deactivates if type != _keyboard_or_game_pad
- Outputs/Return: (none)
- Side effects: Disables SDL relative mouse mode, sets mouse_active=false
- Calls: SDL_SetRelativeMouseMode()

### recenter_mouse
- Signature: `void recenter_mouse(void)`
- Purpose: Recenter mouse on screen when resolution changes
- Inputs: (none)
- Outputs/Return: (none)
- Side effects: Calls MainScreenCenterMouse if mouse_active
- Calls: MainScreenCenterMouse()

### mouse_idle
- Signature: `void mouse_idle(short type)`
- Purpose: Convert raw mouse deltas to camera angles with sensitivity and acceleration applied
- Inputs: `type` ΓÇô input device type (unused; check is implicit via mouse_active)
- Outputs/Return: (none; modifies mouselook_delta)
- Side effects: Consumes snapshot_delta_x/y, applies inversion/sensitivity/acceleration, updates mouselook_delta, zeros deltas
- Calls: TEST_FLAG(), MIX() helper
- Notes:
  - Inverts Y if _inputmod_invert_mouse flag set
  - Base scale: 128/66 degrees per raw unit (tuned for _mouse_accel_none)
  - Multiplies by horizontal/vertical sensitivity from preferences
  - Classic vertical aim reduces pitch sensitivity by 4├ù
  - Supports classic acceleration: multiplies sensitivity by MIX(1, small_factor, accel_scale) based on movement magnitude

### pull_mouselook_delta
- Signature: `fixed_yaw_pitch pull_mouselook_delta()`
- Purpose: Retrieve and consume the frame's accumulated mouselook delta
- Inputs: (none)
- Outputs/Return: Returns mouselook_delta; resets it to {0, 0}
- Side effects: Clears mouselook_delta
- Calls: (none)
- Notes: Called once per frame by game logic to apply camera rotation

### mouse_buttons_become_keypresses
- Signature: `void mouse_buttons_become_keypresses(Uint8* ioKeyMap)`
- Purpose: Map SDL mouse button and scroll states to pseudo-keyscan array entries
- Inputs: `ioKeyMap` ΓÇô array to populate (size >= 407)
- Outputs/Return: (none; modifies ioKeyMap)
- Side effects: Samples SDL mouse state, updates button_mask, consumes scroll wheel delta
- Calls: SDL_GetMouseState()
- Notes:
  - Maps SDL buttons 1ΓÇô5 to keycodes AO_SCANCODE_BASE_MOUSE_BUTTON (400) through 404
  - Scroll up/down map to codes 405/406 (treated as buttons 6/7)
  - Button mask implements "press-to-enable" logic: `button_mask |= ~orig_buttons` requires button release before it can be re-enabled, preventing accidental firing
  - Scroll delta is consumed (zeroed) each call

### hide_cursor / show_cursor
- Signatures: `void hide_cursor(void)` / `void show_cursor(void)`
- Purpose: Hide or display OS mouse cursor
- Inputs: (none)
- Outputs/Return: (none)
- Side effects: SDL_ShowCursor(0) or SDL_ShowCursor(1)
- Calls: SDL_ShowCursor()

### mouse_scroll
- Signature: `void mouse_scroll(bool up)`
- Purpose: Accumulate scroll wheel input
- Inputs: `up` ΓÇô true for scroll up, false for down
- Outputs/Return: (none)
- Side effects: Increments or decrements snapshot_delta_scrollwheel by 1
- Calls: (none)
- Notes: Invoked by SDL event handler; consumed in mouse_buttons_become_keypresses

### mouse_moved
- Signature: `void mouse_moved(int delta_x, int delta_y)`
- Purpose: Accumulate raw mouse motion deltas
- Inputs: `delta_x`, `delta_y` ΓÇô relative movement in pixels
- Outputs/Return: (none)
- Side effects: Adds to snapshot_delta_x/y
- Calls: (none)
- Notes: Called by SDL event handler; consumed in mouse_idle

## Control Flow Notes
**Per-frame input processing:**
1. SDL event pump calls `mouse_moved()` and `mouse_scroll()` to accumulate deltas
2. `mouse_idle()` converts raw deltas to camera angles, applying sensitivity/inversion/acceleration
3. `mouse_buttons_become_keypresses()` samples button state and populates keymap
4. Game logic calls `pull_mouselook_delta()` to retrieve and apply yaw/pitch to player view
5. Deltas reset for next frame

**State transitions:**
- `enter_mouse()` activates mouselook on game start (type != _keyboard_or_game_pad)
- `exit_mouse()` deactivates on pause/exit
- `recenter_mouse()` responds to resolution changes

## External Dependencies
- **SDL2:** SDL_SetHint(), SDL_SetRelativeMouseMode(), SDL_GetMouseState(), SDL_ShowCursor(), type Uint8
- **Engine types:** fixed_angle, fixed_yaw_pitch, _fixed (from world.h); TEST_FLAG macro (from csmacros.h)
- **Screen/input:** MainScreenCenterMouse() (from screen.h); input_preferences global (from preferences.h)
- **SDL constants:** SDL_BUTTON(n), SDL_PRESSED, SDL_RELEASED, SDL_TRUE, SDL_FALSE, SDL_HINT_MOUSE_RELATIVE_MODE_WARP
