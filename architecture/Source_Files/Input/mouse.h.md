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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `fixed_yaw_pitch` | struct (from world.h) | Yaw and pitch rotation angles with fixed-point precision |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `NUM_SDL_REAL_MOUSE_BUTTONS` | #define = 5 | global | Physical mouse button count (1ΓÇô5) |
| `NUM_SDL_MOUSE_BUTTONS` | #define = 7 | global | Virtual button count (includes scroll up/down) |
| `AO_SCANCODE_BASE_MOUSE_BUTTON` | #define = 400 | global | Base pseudo-scancode for mouse button 1 |
| `AO_SCANCODE_MOUSESCROLL_UP` | #define = 405 | global | Pseudo-scancode for scroll-up (button 6) |
| `AO_SCANCODE_MOUSESCROLL_DOWN` | #define = 406 | global | Pseudo-scancode for scroll-down (button 7) |

## Key Functions / Methods

### enter_mouse
- Signature: `void enter_mouse(short type)`
- Purpose: Initialize mouse input for a specified input mode
- Inputs: `type` ΓÇô input mode identifier
- Outputs/Return: None
- Side effects: Initializes mouse driver/event handler

### exit_mouse
- Signature: `void exit_mouse(short type)`
- Purpose: Shutdown mouse input for a specified input mode
- Inputs: `type` ΓÇô input mode identifier
- Outputs/Return: None
- Side effects: Releases mouse resources

### pull_mouselook_delta
- Signature: `fixed_yaw_pitch pull_mouselook_delta()`
- Purpose: Retrieve accumulated mouse-based view rotation since last call
- Inputs: None
- Outputs/Return: `fixed_yaw_pitch` with yaw and pitch deltas (likely zeroed after retrieval)
- Side effects: Likely resets internal delta accumulator

### recenter_mouse
- Signature: `void recenter_mouse(void)`
- Purpose: Reset mouse cursor to screen center (for absolute mouselook mode)
- Inputs: None
- Outputs/Return: None
- Side effects: Sets cursor position, may affect subsequent `pull_mouselook_delta` calls

### mouse_buttons_become_keypresses
- Signature: `void mouse_buttons_become_keypresses(Uint8* ioKeyMap)`
- Purpose: Convert active mouse button states into keyboard scancode map for input polling
- Inputs: `ioKeyMap` ΓÇô keymap array to modify (Uint8 array, likely 512 entries)
- Outputs/Return: None (modifies input array)
- Side effects: Writes pseudo-scancodes (400ΓÇô406) into the keymap

**Notes under this category:**
- `mouse_idle(short type)` ΓÇô Update mouse state for current frame; called during main loop
- `mouse_scroll(bool up)` ΓÇô Register scroll wheel event (up=true ΓåÆ scroll up)
- `mouse_moved(int delta_x, int delta_y)` ΓÇô Register mouse movement delta from SDL event

## Control Flow Notes
Part of the frame input loop: during each frame, the engine calls `mouse_idle()`, then reads mouselook via `pull_mouselook_delta()` to update view angles. Event handlers invoke `mouse_moved()` and `mouse_scroll()` to accumulate input. The button-to-keypress mapping bridges mouse input into the game's key-driven command system (allowing mouse clicks to trigger actions). Cursor centering in `recenter_mouse()` supports first-person relative mouselook.

## External Dependencies
- **Includes:** `world.h` (for `fixed_yaw_pitch` struct definition)
- **External symbols used:** `Uint8` (SDL type for byte)
- **Primitive types:** `short`, `int`, `bool`
