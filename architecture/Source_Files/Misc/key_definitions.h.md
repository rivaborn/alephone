# Source_Files/Misc/key_definitions.h

## File Purpose
Defines keyboard input mappings and configuration structures for the game engine's input system. Provides a standard keyboard layout that maps SDL scancodes to player action flags, along with data structures for advanced key binding features like blacklisting and special flag handling.

## Core Responsibilities
- Define the default (standard) keyboard-to-action mapping used by the input system
- Provide data structures for tracking blacklisted key combinations
- Support special flag definitions with persistence behavior
- Supply a reference key layout that matches the UI dialog order in the key setup dialog
- Enable multi-keyboard setup configurations (standard, left-handed, PowerBook variants)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `blacklist_data` | struct | Stores offset/mask pairs to identify and blacklist problematic key combinations |
| `special_flag_data` | struct | Defines special action flags with type (double/latched) and persistence timing |
| `key_definition` | struct | Maps an SDL scancode to a player action flag |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `standard_key_definitions` | `struct key_definition[]` | static | Default keyboard mapping; defines 21 key bindings for movement, aiming, triggers, modifiers, and UI actions |

## Key Functions / Methods
None. This is a data-definition header with no executable functions.

## Control Flow Notes
This file is part of the **input handling subsystem** and is intended to be included only by `vbl.c` (the vertical blank/frame update handler). The `standard_key_definitions` array is loaded during initialization and serves as a reference configuration when the player selects the standard keyboard layout in settings. The action flags map into the bitfield structure defined in `player.h`, where they are aggregated each frame into an action_flags word and passed to `process_action_flags()` for player physics updates.

## External Dependencies
- **Includes:** `interface.h`, `player.h`
- **External symbols used:**
  - `SDL_Scancode` enum (SDL library)
  - Action flag macros from `player.h`: `_moving_forward`, `_moving_backward`, `_turning_left`, `_turning_right`, `_sidestepping_left`, `_sidestepping_right`, `_looking_left`, `_looking_right`, `_looking_up`, `_looking_down`, `_looking_center`, `_cycle_weapons_backward`, `_cycle_weapons_forward`, `_left_trigger_state`, `_right_trigger_state`, `_sidestep_dont_turn`, `_run_dont_walk`, `_look_dont_turn`, `_action_trigger_state`, `_toggle_map`, `_microphone_button`

## Notes
- The file header comment notes this was authored 1994 and restricts inclusion to `vbl.c`, suggesting tight coupling with the main input polling loop.
- The standard layout uses WASD-adjacent keys (Z/X for strafe, A/S for look, D/C/V for vertical look) and keypad for movementΓÇöa common 1990s FPS layout.
- The `NUMBER_OF_STANDARD_KEY_DEFINITIONS` macro uses sizeof arithmetic for robust array-length calculation.
- No action mappings for pause, save, or menu navigation appear here; those are likely handled elsewhere in the UI system.
