# Source_Files/Input/joystick.h

## File Purpose
Header file declaring the joystick input subsystem interface for Aleph One. Defines scancode offsets for mapping SDL controller buttons and axes into the engine's keymap array, and declares lifecycle and event-handling functions for joystick input management.

## Core Responsibilities
- Define scancode constants for mapping SDL joystick/controller inputs to the engine's keymap
- Manage joystick subsystem initialization and lifecycle (enter/exit)
- Convert joystick button states and axis movements into keyboard events
- Handle joystick device hotplug events (added/removed)
- Process joystick input during frame updates

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AO_SCANCODE_BASE_JOYSTICK_BUTTON` | `const int` | global | Base scancode offset (415) for joystick button mappings |
| `AO_SCANCODE_BASE_JOYSTICK_AXIS_POSITIVE` | `const int` | global | Scancode offset for positive axis movements |
| `AO_SCANCODE_BASE_JOYSTICK_AXIS_NEGATIVE` | `const int` | global | Scancode offset for negative axis movements |
| `NUM_SDL_JOYSTICK_BUTTONS` | `const int` | global | Total count of mappable joystick inputs (buttons + axes) |
| `AO_SCANCODE_JOYSTICK_ESCAPE` | `const SDL_Scancode` | global | Maps START button to engine escape scancode |

## Key Functions / Methods

### initialize_joystick
- **Signature:** `void initialize_joystick(void)`
- **Purpose:** Initialize the joystick subsystem
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes SDL joystick state; likely registers event handlers
- **Calls:** Not visible in this file
- **Notes:** Called once at engine startup

### enter_joystick / exit_joystick
- **Signature:** `void enter_joystick(void)` / `void exit_joystick(void)`
- **Purpose:** Scope-based lifecycle management for joystick input (e.g., pause menu)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Enables/disables joystick input processing
- **Notes:** Likely paired for nested scopes

### joystick_buttons_become_keypresses
- **Signature:** `void joystick_buttons_become_keypresses(Uint8* ioKeyMap)`
- **Purpose:** Poll current joystick state and update the engine's keymap array
- **Inputs:** `ioKeyMap` ΓÇô pointer to keymap array to modify
- **Outputs/Return:** None (modifies `ioKeyMap` in-place)
- **Side effects:** Reads SDL joystick state; updates input array
- **Calls:** SDL joystick polling functions (not visible)
- **Notes:** Likely called once per frame; "become" implies translation/polling rather than event-driven update

### process_joystick_axes
- **Signature:** `int process_joystick_axes(int flags)`
- **Purpose:** Process joystick axis input with optional flags (e.g., dead-zone filtering)
- **Inputs:** `flags` ΓÇô control flags for processing behavior
- **Outputs/Return:** Status or result code (semantics not inferable)
- **Side effects:** May update internal axis state
- **Notes:** Separate from button processing; likely handles analog stick/trigger values

### joystick_axis_moved
- **Signature:** `void joystick_axis_moved(int instance_id, int axis, int value)`
- **Purpose:** Event handler for axis movement
- **Inputs:** `instance_id` (joystick device), `axis` (axis index), `value` (normalized or raw axis value)
- **Outputs/Return:** None
- **Side effects:** Updates internal joystick state
- **Calls:** Not visible in this file

### joystick_button_pressed
- **Signature:** `void joystick_button_pressed(int instance_id, int button, bool down)`
- **Purpose:** Event handler for button press/release
- **Inputs:** `instance_id`, `button` (button index), `down` (true = pressed, false = released)
- **Outputs/Return:** None
- **Side effects:** Updates keymap or button state

### joystick_added / joystick_removed
- **Signature:** `void joystick_added(int device_index)` / `bool joystick_removed(int instance_id)`
- **Purpose:** Handle joystick hotplug events
- **Inputs:** Device/instance ID
- **Outputs/Return:** `joystick_removed` returns success/failure bool
- **Side effects:** Register/unregister joystick device in internal tracking

## Control Flow Notes
Event-driven with polling support. Joystick input flows via: initialization ΓåÆ per-frame polling (`joystick_buttons_become_keypresses`, `process_joystick_axes`) + event callbacks (axis/button/hotplug) ΓåÆ scancodes written to keymap ΓåÆ engine input layer processes keymap.

## External Dependencies
- `cstypes.h` ΓÇô type aliases (`Uint8`)
- **SDL2** ΓÇô `SDL_Scancode`, `SDL_CONTROLLER_*` constants (controller button and axis count macros)
