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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `AxisInfo` | struct | Maps SDL controller axis index to action binding index, pitch/yaw flag, and sign |
| `active_instances` | `std::map<int, SDL_GameController*>` | Tracks connected controllers by SDL instance ID |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `joystick_active` | int | global | Enable/disable joystick input (e.g., on focus loss) |
| `axis_values` | int[SDL_CONTROLLER_AXIS_MAX] | file-static | Current raw axis values; Y-axes flipped for movement convention |
| `button_values` | bool[NUM_SDL_JOYSTICK_BUTTONS] | file-static | Current button states and axis threshold crossings |
| `axis_mappings` | const vector<AxisInfo> | file-static | Hardcoded mappings: axes 2,3 ΓåÆ yaw; 8,9 ΓåÆ pitch |
| `active_instances` | static map | file-static | SDL controller handle registry |

## Key Functions / Methods

### initialize_joystick
- **Signature:** `void initialize_joystick(void)`
- **Purpose:** Enable SDL game controller events and open all currently connected devices
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Populates `active_instances` map; enables SDL event generation
- **Calls:** `SDL_GameControllerEventState()`, `SDL_NumJoysticks()`, `joystick_added()`
- **Notes:** Called once on engine startup

### enter_joystick / exit_joystick
- **Signature:** `void enter_joystick(void)` / `void exit_joystick(void)`
- **Purpose:** Enable/disable joystick input (e.g., on window focus)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `joystick_active` flag
- **Calls:** None
- **Notes:** Input is buffered but discarded if inactive

### joystick_added
- **Signature:** `void joystick_added(int device_index)`
- **Purpose:** Open a newly connected controller if it has SDL database mapping
- **Inputs:** `device_index` (SDL joystick device index)
- **Outputs/Return:** None
- **Side effects:** Adds entry to `active_instances`; logs warning if unmapped
- **Calls:** `SDL_IsGameController()`, `SDL_GameControllerOpen()`, `SDL_JoystickInstanceID()`, `logWarning()`
- **Notes:** Unmapped joysticks are rejected; requires SDL mapping string in database

### joystick_removed
- **Signature:** `bool joystick_removed(int instance_id)`
- **Purpose:** Close and deregister a disconnected controller
- **Inputs:** `instance_id` (SDL joystick instance ID)
- **Outputs/Return:** true if removed, false if not found
- **Side effects:** Closes SDL handle; erases from `active_instances`
- **Calls:** `SDL_GameControllerClose()`
- **Notes:** Called on SDL controller removal event

### joystick_axis_moved
- **Signature:** `void joystick_axis_moved(int instance_id, int axis, int value)`
- **Purpose:** Update raw axis value and derive discrete button states for threshold crossing
- **Inputs:** `instance_id` (unusedΓÇösingle global axis state), `axis` (SDL_CONTROLLER_AXIS_*), `value` [-32768, 32767]
- **Outputs/Return:** None
- **Side effects:** Updates `axis_values[]` and `button_values[]` for axis positive/negative thresholds (┬▒16384)
- **Calls:** None
- **Notes:** Y-axes negated to match Marathon movement convention. Threshold-derived buttons allow hybrid analog+button input.

### joystick_button_pressed
- **Signature:** `void joystick_button_pressed(int instance_id, int button, bool down)`
- **Purpose:** Update button state array on button press/release
- **Inputs:** `instance_id` (unused), `button` (SDL_CONTROLLER_BUTTON_*), `down` (pressed?)
- **Outputs/Return:** None
- **Side effects:** Updates `button_values[button]`
- **Calls:** None
- **Notes:** Bounds check: only updates if button < NUM_SDL_JOYSTICK_BUTTONS

### axis_mapped_to_action
- **Signature:** `static int axis_mapped_to_action(int action, bool* negative)`
- **Purpose:** Query key binding configuration to find which axis (if any) is bound to an action
- **Inputs:** `action` (key binding index, e.g., for yaw/pitch), `negative` (output: is binding for negative direction?)
- **Outputs/Return:** Axis index [0, SDL_CONTROLLER_AXIS_MAX) or -1 if unbound
- **Side effects:** Writes to `negative` output parameter
- **Calls:** Accesses `input_preferences->key_bindings[action]`
- **Notes:** Searches codeset for scancodes in joystick axis ranges; translates to axis index

### joystick_buttons_become_keypresses
- **Signature:** `void joystick_buttons_become_keypresses(Uint8* ioKeyMap)`
- **Purpose:** Convert button state array into keymap for engine input system; avoid buttons mapped to analog aiming
- **Inputs:** `ioKeyMap` (pointer to key state array, indexed by SDL_Scancode)
- **Outputs/Return:** None (modifies keymap in-place)
- **Side effects:** Updates keymap entries for all button scancodes
- **Calls:** `axis_mapped_to_action()`
- **Notes:** Early exit if joystick inactive or no controllers connected. Excludes analog-bound axes from button reporting if `controller_analog` enabled.

### process_joystick_axes
- **Signature:** `int process_joystick_axes(int flags)`
- **Purpose:** Convert analog stick deltas (with deadzone/sensitivity) into yaw/pitch deltas and feed to aim processor
- **Inputs:** `flags` (current action flags)
- **Outputs/Return:** Updated action flags with yaw/pitch deltas encoded
- **Side effects:** None (flags modified return value only)
- **Calls:** `axis_mapped_to_action()`, `process_aim_input()`
- **Notes:** 
  - Early exit if joystick inactive, no controllers, or `controller_analog` disabled
  - Applies per-axis deadzone (clamps values below threshold to 0)
  - Normalizes by sensitivity preference and SDL range (┬▒32767)
  - Angle conversion: 768/63 radians per unit norm
  - Applies axis negation and vertical aim inversion separately
  - Returns unchanged flags if no aim deltas generated

## Control Flow Notes
- **Startup:** `initialize_joystick()` called once; SDL events enabled
- **Per-frame:** 
  - `joystick_buttons_become_keypresses()` converts buffered button states to keymap
  - `process_joystick_axes()` converts buffered axis states to aim deltas
- **Asynchronous:** `joystick_axis_moved()`, `joystick_button_pressed()`, `joystick_added()`, `joystick_removed()` called by SDL event loop
- **Focus management:** `enter_joystick()` / `exit_joystick()` toggle input buffering without dropping events

## External Dependencies
- **SDL2:** Controller initialization, event state, handle management, GUID lookup
- **player.h:** `mask_in_absolute_positioning_information` (macro/function); `process_aim_input()` function; action flag bit definitions
- **preferences.h:** `input_preferences` global; key binding map, controller deadzone/sensitivity settings, mode flags
- **joystick.h:** Scancode offset constants (AO_SCANCODE_BASE_*), button count macro
- **Logging.h:** `logWarning()` macro for unmapped controller warnings
