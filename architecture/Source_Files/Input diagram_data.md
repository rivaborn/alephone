# Source_Files/Input/joystick.h
## File Purpose
Header file declaring the joystick input subsystem interface for Aleph One. Defines scancode offsets for mapping SDL controller buttons and axes into the engine's keymap array, and declares lifecycle and event-handling functions for joystick input management.

## Core Responsibilities
- Define scancode constants for mapping SDL joystick/controller inputs to the engine's keymap
- Manage joystick subsystem initialization and lifecycle (enter/exit)
- Convert joystick button states and axis movements into keyboard events
- Handle joystick device hotplug events (added/removed)
- Process joystick input during frame updates

## External Dependencies
- `cstypes.h` ΓÇô type aliases (`Uint8`)
- **SDL2** ΓÇô `SDL_Scancode`, `SDL_CONTROLLER_*` constants (controller button and axis count macros)

# Source_Files/Input/joystick_sdl.cpp
## File Purpose
SDL2-based gamepad input handler for the Aleph One engine. Manages game controller device lifecycle, translates button/axis input into key events and analog aiming data, and applies user-configured sensitivity and deadzone settings.

## Core Responsibilities
- Initialize SDL2 game controller event system and enumerate connected devices
- Track active controllers by SDL instance ID; open/close controllers on device connect/disconnect
- Buffer current axis and button state; convert analog triggers into discrete button events
- Map controller input to scancode-based keymaps for the input system
- Process analog stick deltas with deadzone/sensitivity/inversion for aim targeting
- Respect controller analog mode preference and key binding configuration

## External Dependencies
- **SDL2:** Controller initialization, event state, handle management, GUID lookup
- **player.h:** `mask_in_absolute_positioning_information` (macro/function); `process_aim_input()` function; action flag bit definitions
- **preferences.h:** `input_preferences` global; key binding map, controller deadzone/sensitivity settings, mode flags
- **joystick.h:** Scancode offset constants (AO_SCANCODE_BASE_*), button count macro
- **Logging.h:** `logWarning()` macro for unmapped controller warnings

# Source_Files/Input/mouse.h
## File Purpose
Input handling interface for mouse control in the Aleph One game engine. Provides functions to initialize/manage mouse input, retrieve mouselook deltas, and map mouse buttons to keyboard input via SDL. Part of the Marathon engine's input subsystem.

## Core Responsibilities
- Initialize and shutdown mouse input for different input modes (`enter_mouse`, `exit_mouse`)
- Query accumulated mouselook rotation per frame (`pull_mouselook_delta`)
- Maintain mouse state during idle frames and recenter (`mouse_idle`, `recenter_mouse`)
- Map SDL mouse buttons to keyboard scancodes for game input (`mouse_buttons_become_keypresses`)
- Handle scroll wheel input as virtual buttons (`mouse_scroll`)
- Track raw mouse movement deltas (`mouse_moved`)

## External Dependencies
- **Includes:** `world.h` (for `fixed_yaw_pitch` struct definition)
- **External symbols used:** `Uint8` (SDL type for byte)
- **Primitive types:** `short`, `int`, `bool`

# Source_Files/Input/mouse_sdl.cpp
## File Purpose
Implements SDL-specific mouse input handling for in-game mouselook, including sensitivity/acceleration control, mouse button emulation as keypresses, scroll wheel input, and cursor management.

## Core Responsibilities
- Initialize and shutdown in-game mouse input mode with relative mouse tracking
- Calculate mouselook yaw/pitch deltas from raw mouse movements with configurable sensitivity, inversion, and acceleration
- Convert SDL mouse buttons to pseudo-keyscan codes for the input system
- Map scroll wheel movements to button-like input
- Manage cursor visibility and recenter on screen size changes

## External Dependencies
- **SDL2:** SDL_SetHint(), SDL_SetRelativeMouseMode(), SDL_GetMouseState(), SDL_ShowCursor(), type Uint8
- **Engine types:** fixed_angle, fixed_yaw_pitch, _fixed (from world.h); TEST_FLAG macro (from csmacros.h)
- **Screen/input:** MainScreenCenterMouse() (from screen.h); input_preferences global (from preferences.h)
- **SDL constants:** SDL_BUTTON(n), SDL_PRESSED, SDL_RELEASED, SDL_TRUE, SDL_FALSE, SDL_HINT_MOUSE_RELATIVE_MODE_WARP


