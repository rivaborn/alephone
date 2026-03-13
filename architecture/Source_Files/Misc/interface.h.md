# Source_Files/Misc/interface.h

## File Purpose
Header file declaring the main interface/state management system for the Marathon game engine (Aleph One). Provides constants, data structures, and function prototypes for game state transitions, UI/menu handling, input processing, shape/texture management, and system-level operations.

## Core Responsibilities
- Define game state machine (intro screens, menus, game-in-progress, etc.) and controller types
- Declare UI/menu functions (main menu rendering, dialogs, preferences)
- Manage shape/collection loading, caching, and rendering metadata
- Provide game state query and transition functions
- Handle input processing (keyboard, mouse, replay, network)
- Interface with physics, networking, and save/load systems
- Expose resource file and resource specifier utilities

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_information_data` | struct | Metadata for individual shapes: mirror flags, lighting intensity, world-space bounds, hotpoint |
| `shape_animation_data` | struct | Animation parameters: frame counts, tick timings, sound indices, transfer mode, loop behavior, shape index array |

## Global / File-Static State
None.

## Key Functions / Methods

### Game State Management

#### `set_game_state` / `get_game_state`
- Purpose: Set or query the current game state (menu, in-progress, loading, etc.)
- Inputs: `short new_state` (for set); none (for get)
- Outputs/Return: void (set); `short` current state (get)
- Notes: States include display modes, game-in-progress, level changes, and quit conditions

#### `initialize_game_state`
- Purpose: Initialize the game state machine to default
- Inputs: none
- Outputs/Return: void
- Side effects: Sets up initial game state

#### `force_game_state_change` / `idle_game_state`
- Purpose: Force immediate state change; advance idle animation/logic
- Inputs: none (force); `uint64_t time` (idle)
- Outputs/Return: void (force); `bool` indicating completion (idle)

#### `player_controlling_game`
- Purpose: Query whether human player or AI/replay is in control
- Inputs: none
- Outputs/Return: `bool`

### Shape/Texture/Collection Management

#### `get_shape_descriptors`
- Purpose: Retrieve shape descriptor array for a given shape type
- Inputs: `short shape_type`, `shape_descriptor *buffer`
- Outputs/Return: `short` count of descriptors
- Calls: Collection loading system

#### `extended_get_shape_bitmap_and_shading_table` (macro: `get_shape_bitmap_and_shading_table`)
- Purpose: Retrieve bitmap and shading table for rendering a shape
- Inputs: collection code, low-level shape index, shading mode
- Outputs/Return: `bitmap_definition **`, `void **shading_tables`
- Notes: Macro wraps full function; used for sprite/texture rendering

#### `get_shape_animation_data` / `get_shape_information`
- Purpose: Return animation or metadata struct for a shape
- Inputs: `shape_descriptor` or collection/shape indices
- Outputs/Return: `shape_animation_data *` or `shape_information_data *`

#### `mark_collection` / `load_collections` / `unload_all_collections`
- Purpose: Mark collections for load/unload; perform batch loading; unload all
- Inputs: collection code, `bool loading`; `bool with_progress_bar`, `bool is_opengl`
- Outputs/Return: void
- Side effects: I/O, memory allocation
- Notes: Collections are groups of shapes; load/unload manages VRAM or system memory

#### `is_collection_present` / `get_number_of_collection_frames` / `get_number_of_collection_bitmaps`
- Purpose: Query collection metadata
- Inputs: collection index
- Outputs/Return: `bool`, `short`

### Input & Keyboard

#### `process_action_flags` / `parse_keymap`
- Purpose: Process accumulated input actions; parse keyboard state
- Inputs: player ID, action flags array, count (actions); none (keymap)
- Outputs/Return: void (actions); `uint32` parsed keymap (keymap)
- Calls: VBL subsystem

#### `set_keyboard_controller_status` / `pause_keyboard_controller`
- Purpose: Enable/disable or pause keyboard input
- Inputs: `bool active`
- Outputs/Return: void (set); `bool` (get status)

### Game Flow

#### `pause_game` / `resume_game`
- Purpose: Pause or resume game execution
- Inputs: none
- Outputs/Return: void

#### `set_change_level_destination` / `check_level_change`
- Purpose: Request level change; check if level change is pending
- Inputs: `short level_number` (set); none (check)
- Outputs/Return: void (set); `bool` (check)

#### `revert_game` / `save_game` / `load_game` / `restart_game`
- Purpose: Revert to last checkpoint, save/load game state, restart level
- Inputs: various file/load mode parameters
- Outputs/Return: `bool` success

### UI & Dialogs

#### `display_main_menu` / `draw_main_menu` / `draw_menu_button`
- Purpose: Show/render main menu and menu items
- Inputs: index (for button); base_id for buffer management
- Outputs/Return: void
- Side effects: Screen rendering, event handling

#### `do_menu_item_command`
- Purpose: Execute command associated with menu selection
- Inputs: `short menu_id`, `short menu_item`, `bool cheat`
- Outputs/Return: void

#### `do_preferences` / `handle_preferences_dialog` / `handle_load_game` / `handle_save_game`
- Purpose: Open and handle various game dialogs
- Inputs: none (most)
- Outputs/Return: `bool` success or user action
- Side effects: Modal dialogs, file I/O

#### `portable_process_screen_click`
- Purpose: Handle mouse clicks in-game or in UI
- Inputs: `short x, y`, `bool cheatkeys_down`
- Outputs/Return: void

### Miscellaneous

#### `ResetFieldOfView` / `ReloadViewContext`
- Purpose: Reset FOV to player state; reload rendering context
- Inputs: none
- Outputs/Return: void
- Notes: Supports extravision toggle; prevents rendering artifacts after level load

#### `dont_switch_to_new_weapon` / `dont_auto_recenter` / `is_player_behavior_standard`
- Purpose: Query or standardize player behavior modifiers (for net/film compatibility)
- Inputs: none
- Outputs/Return: `bool`

## Control Flow Notes
- **Initialization**: `initialize_game_state()` sets up the state machine; `load_collections()` preloads graphics
- **Main loop**: `idle_game_state()` advances UI/menu logic; `process_action_flags()` and VBL subsystem handle input/physics
- **State transitions**: `set_game_state()` manages transitions between menu, game, dialogs, and cinematic screens; `check_level_change()` gates level transitions
- **Shutdown**: `unload_all_collections()` frees resources; game state returns to menu or quit
- **Multiplayer/Recording**: Network and replay systems hook into input processing and state transitions via dedicated functions

## External Dependencies
- **cseries.h**: Platform abstractions, data types, utility macros
- **shape_descriptors.h**: Shape descriptor encoding (collection/shape/CLUT bits) and collection constants
- **FileSpecifier**: Forward-declared; resource file abstraction
- **OpenedResourceFile**: Forward-declared; resource loading
- **InfoTree**: Forward-declared; XML parsing for MML (Marathon Markup Language) config
- Implied: physics.c, vbl.c (VBL/frame timing), game_window.c, network.c, game_dialogs.c, shapes.c (shape loading), preprocess_map_mac.c (save/load)
