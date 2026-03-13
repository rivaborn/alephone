# Subsystem Overview: Input

## Purpose
The Input subsystem abstracts SDL2 input devices (keyboard, mouse, gamepad) and converts hardware input into the engine's scancode-based keymap format. It manages device lifecycle, applies user-configured sensitivity and deadzone settings, and bridges raw input events to game actions.

## Key Files
| File | Role |
|------|------|
| `joystick.h` | Declares joystick interface; defines scancode offset constants for controller button/axis mapping |
| `joystick_sdl.cpp` | SDL2 game controller implementation; manages device lifecycle, translates buttons/axes to keyscan codes and analog aim data |
| `mouse.h` | Declares mouse input interface; defines mouselook and button handling functions |
| `mouse_sdl.cpp` | SDL2 mouse implementation; calculates mouselook deltas with sensitivity/acceleration; maps buttons and scroll wheel |

## Core Responsibilities
- Enumerate and manage SDL2 input device lifecycle (controllers, mouse hotplug events)
- Map hardware inputs (buttons, analog sticks, axes, scroll wheel) to scancode-based keymaps
- Convert continuous analog values (mouse deltas, controller sticks) to discrete key events and aim targeting data
- Apply user-configured input preferences (sensitivity, deadzone, inversion, key bindings)
- Handle relative mouse mode for in-game mouselook with acceleration curve
- Process frame-by-frame input state buffering and device event polling

## Key Interfaces & Data Flow
**Exposes to other subsystems:**
- Scancode offset constants for controller input mapping into keymap array
- Initialization/shutdown functions (`enter_mouse`, `exit_mouse`, joystick equivalents)
- Mouselook delta query function (`pull_mouselook_delta`)
- Frame update processing to buffer and convert input events

**Consumes from other subsystems:**
- **SDL2:** Controller/mouse device state, events, relative motion tracking, button/axis constants
- **preferences.h:** `input_preferences` global (key bindings, sensitivity, deadzone, inversion, analog mode flags)
- **player.h:** Action flag definitions, `process_aim_input()` function, positioning macros
- **world.h:** `fixed_yaw_pitch`, `fixed_angle` types for mouselook calculations
- **screen.h:** `MainScreenCenterMouse()` for cursor recentering

## Runtime Role
- **Init:** `enter_mouse()` and joystick initialization enable SDL relative mouse mode and enumerate connected controllers
- **Frame:** Poll SDL events for device hotplug and button/axis state; buffer current input and convert to keyscan codes; calculate mouselook deltas and pass analog aim input to player subsystem
- **Shutdown:** `exit_mouse()` and joystick exit disable relative mouse mode and release controller handles

## Notable Implementation Details
- Joystick inputs routed through `AO_SCANCODE_BASE_*` offset constants into engine keymap array
- Analog triggers converted to discrete button events based on threshold
- Controller device management tracks active controllers by SDL instance ID; devices open/close on hotplug
- Mouse mouselook optionally applies acceleration curve with configurable sensitivity and inversion
- Scroll wheel mapped as virtual button input (discrete events)
- Deadzone filtering applied to analog stick input before aim targeting calculation
