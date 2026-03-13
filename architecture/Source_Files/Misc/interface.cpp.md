# Source_Files/Misc/interface.cpp

## File Purpose
Implements the Marathon/Aleph One game state machine and main interface loop, managing transitions between intro screens, menus, gameplay, replays, and game completion sequences. Handles screen rendering, fading effects, user input processing, and coordinates high-level game lifecycle events including single-player, network multiplayer, and saved game restoration.

## Core Responsibilities
- Game state machine management and state transitions (intro ΓåÆ menu ΓåÆ gameplay ΓåÆ epilogue)
- Interface fade effects and color table management for screen transitions
- Main menu rendering, button interaction, and menu command dispatch
- Game initialization, startup, pausing, resuming, and cleanup
- Save game and replay file loading and management
- Network game setup and multiplayer game initiation
- MPEG movie playback with audio/video synchronization
- Screen mode changes and display updates based on game context
- Coordinate level changes and chapter screen display during gameplay

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `game_state` | struct | Tracks current game state, phase counter, player type, flags |
| `screen_data` | struct | Defines screen sequence: base ID, count, duration |
| `recording_version` | enum | Compatibility versioning for replay/recording formats |
| `steam_workshop_uploader_ui_data` | struct | Steam workshop integration for content upload |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `game_state` | `struct game_state` | static | Current game state machine state and phase |
| `introduction_sound` | `std::shared_ptr<SoundPlayer>` | static | Intro screen background audio |
| `DraggedReplayFile` | `FileSpecifier` | static | Replay file path for open-in-place loading |
| `interface_fade_in_progress` | bool | static | Tracks if color table fade is active |
| `current_picture_clut` | `color_table*` | static | Active color lookup table for current screen |
| `animated_color_table` | `color_table*` | static | Scratch buffer for fade animation |
| `current_picture_clut_depth` | short | static | Bit depth when CLUT was created |
| `display_screens[]` | `screen_data[]` | static | Screen sequence data for M2/Infinity |
| `m1_display_screens[]` | `screen_data[]` | static | Screen sequence data for Marathon 1 mode |

## Key Functions / Methods

### initialize_game_state
- **Signature:** `void initialize_game_state(void)`
- **Purpose:** Set up initial game state machine; determine whether to show intro or main menu based on preferences and startup mode
- **Inputs:** None (uses `shell_options` global)
- **Outputs/Return:** None; modifies `game_state` static variable
- **Side effects:** Disables menus, loads Lua (if enabled), initializes `game_state.state` to `_display_intro_screens` or calls `display_main_menu()`
- **Calls:** `toggle_menus()`, `alert_user()`, `expand_app_variables()`, `display_main_menu()`, `display_introduction()`
- **Notes:** Called once at application startup; handles insecure Lua warning

### set_game_state
- **Signature:** `void set_game_state(short new_state)`
- **Purpose:** Transition to a new game state with validation rules (e.g., defer certain transitions to idle time)
- **Inputs:** `new_state` - target game state constant
- **Outputs/Return:** None; modifies `game_state.state`
- **Side effects:** May set `game_state.phase = 0` to trigger deferred processing; calls `finish_game()` on quit transitions
- **Calls:** `finish_game()`, `display_quit_screens()`
- **Notes:** Enforces safe state transitions; defers level/demo changes to idle time to avoid interrupt-level issues

### idle_game_state
- **Signature:** `bool idle_game_state(uint64_t time)`
- **Purpose:** Main update function for non-gameplay states (menus, screens); called every frame during intro/menu phases
- **Inputs:** `time` - elapsed time in machine ticks
- **Outputs/Return:** bool - true if state remains the same
- **Side effects:** Advances screens on timeout, handles fades, processes deferred state changes, updates music
- **Calls:** `next_game_screen()`, `update_interface_fades()`, `display_screen()`, `Music::instance()` methods
- **Notes:** Implements timeout-based screen advancement and fade completion

### begin_game
- **Signature:** `static bool begin_game(short user, bool cheat)`
- **Purpose:** Initialize a game (new, demo, or replay) and transition to gameplay
- **Inputs:** `user` - `_single_player`, `_network_player`, `_replay`, `_demo`, or `_replay_from_file`; `cheat` - enable cheats flag
- **Outputs/Return:** bool - success/failure
- **Side effects:** Loads map, sets up players, initializes recording/playback, starts Lua scripts, transitions game state to `_game_in_progress`
- **Calls:** `load_game_from_file()`, `new_game()`, `start_game()`, `entering_map()`, `RunLuaScript()`, `start_recording()`, `start_replay()`
- **Notes:** Handles single-player, multiplayer, demo, and replay modes; sets up generalized player starts

### start_game
- **Signature:** `static void start_game(short user, bool changing_level)`
- **Purpose:** Begin actual gameplay after all setup is complete
- **Inputs:** `user` - game controller type; `changing_level` - true if transitioning between levels
- **Outputs/Return:** None
- **Side effects:** Hides cursor, updates HUD scripts, sets game state to `_game_in_progress`, resumes audio
- **Calls:** `hide_cursor()`, `resume_game()`, `RunLuaHUDScript()`, `reset_screen()`
- **Notes:** Called after map loading and player initialization are done

### finish_game
- **Signature:** `static void finish_game(bool return_to_main_menu)`
- **Purpose:** Clean up after game (single-player, network, or replay) and optionally return to menu
- **Inputs:** `return_to_main_menu` - show menu after cleanup
- **Outputs/Return:** None
- **Side effects:** Stops recording/replay, unloads collections and sounds, fades out music, cleans up HUD, exits networking if applicable, leaves map, displays stats or main menu
- **Calls:** `stop_recording()`, `stop_replay()`, `unload_all_collections()`, `Music::instance()->Fade()`, `leaving_map()`, `display_net_game_stats()`, `display_main_menu()`, `NetUnSync()`
- **Notes:** Handles both normal exit and replay/network cleanup; may export level if in editor mode

### load_and_start_game
- **Signature:** `bool load_and_start_game(FileSpecifier& File)`
- **Purpose:** Load a saved game and optionally start it as single-player or networked game
- **Inputs:** `File` - path to saved game file
- **Outputs/Return:** bool - success
- **Side effects:** Loads saved game WAD, asks user for single/multiplayer mode, sets up network if applicable, initializes player starts
- **Calls:** `load_game_from_file()`, `should_restore_game_networked()`, `network_gather()`, `join_networked_resume_game()`, `make_restored_game_relevant()`, `start_recording()`
- **Notes:** Implements generalized game startup; coordinates with network code for multiplayer resumption

### show_movie
- **Signature:** `void show_movie(short index)`
- **Purpose:** Play an MPEG video with synchronized audio using pl_mpeg decoder and libyuv scaling
- **Inputs:** `index` - movie index (usually 0 for intro movie)
- **Outputs/Return:** None
- **Side effects:** Changes screen mode, decodes MPEG frames, streams audio, renders to screen until completion or user input
- **Calls:** `GetLevelMovie()`, `plm_create_with_filename()`, `libyuv::I420Scale()`, `OpenALManager::Get()->PlayStream()`, `OGL_Blitter` or SDL blit
- **Notes:** Uses tuples for callback userdata; supports both OpenGL and SDL rendering paths; handles I420 to RGBA/ABGR conversion

### interface_fade_out
- **Signature:** `void interface_fade_out(short pict_resource_number, bool fade_music)`
- **Purpose:** Fade out current screen picture and optionally music before transitioning
- **Inputs:** `pict_resource_number` - resource ID for recalculating CLUT if needed; `fade_music` - also fade out music
- **Outputs/Return:** None
- **Side effects:** Checks/recalculates color table if bit depth changed, fades music, pauses audio, resets palette to black, frees color table
- **Calls:** `Music::instance()->Fade()`, `full_fade()`, `paint_window_black()`
- **Notes:** Precondition: `current_picture_clut` must be non-NULL; handles bit depth changes mid-fade

### handle_interface_menu_screen_click
- **Signature:** `static void handle_interface_menu_screen_click(short x, short y, bool cheatkeys_down)`
- **Purpose:** Process mouse click on main menu button, with visual feedback during drag
- **Inputs:** `x`, `y` - screen coordinates; `cheatkeys_down` - cheat keys enabled
- **Outputs/Return:** None
- **Side effects:** Draws button pressed state, polls mouse movement, executes menu command if released over button
- **Calls:** `get_interface_rectangle()`, `point_in_rectangle()`, `draw_button()`, `do_menu_item_command()`, SDL event polling
- **Notes:** Modal loop with real-time redraw during drag; supports mobile/gamepad input via SDL event types

### pause_game / resume_game
- **Signature:** `void pause_game(void)` and `void resume_game(void)`
- **Purpose:** Pause/resume gameplay (stop input, show cursor, pause audio)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Toggles keyboard controller, cursor visibility, audio pause status, revalidates world window
- **Calls:** `set_keyboard_controller_status()`, `show_cursor()`, `hide_cursor()`, `OpenALManager::Get()->Pause()`

### player_controlling_game
- **Signature:** `bool player_controlling_game(void)`
- **Purpose:** Query whether the player has active control (vs. demo, replay, menus)
- **Inputs:** None
- **Outputs/Return:** bool - true if user is playing (single or network) in active gameplay state
- **Notes:** Used to gate input processing and AI

### make_restored_game_relevant
- **Signature:** `static bool make_restored_game_relevant(bool inNetgame, const player_start_data* inStartArray, short inStartCount)`
- **Purpose:** Restore saved game state, synchronize players with starts, initialize map scripting
- **Inputs:** `inNetgame` - true for multiplayer restore; `inStartArray`, `inStartCount` - player starts
- **Outputs/Return:** bool - success
- **Side effects:** Runs level scripts, sets random seed, synchronizes players, calls `entering_map()`, resets motion sensor
- **Calls:** `RunLuaScript()`, `set_random_seed()`, `synchronize_players_with_starts()`, `entering_map()`, `reset_motion_sensor()`, `clean_up_after_failed_game()`
- **Notes:** Shared code for single-player and network game resumption; handles difficulty level and behavior modifiers

## Control Flow Notes

**State Machine States:**
- **Menu/Display States** (`_display_intro_screens`, `_display_main_menu`, `_display_chapter_heading`, `_display_prologue`, `_display_epilogue`, `_display_credits`): Non-gameplay; advanced by `idle_game_state()` based on timeout and user input
- **Gameplay State** (`_game_in_progress`): Active game loop; can transition to `_switch_demo`, `_revert_game`, `_change_level`, `_quit_game`, `_close_game`
- **Network/Dialog States** (`_displaying_network_game_dialogs`): Used during gather/join and stats display

**Frame Loop Integration:**
- `idle_game_state()` called from main loop when not in `_game_in_progress`
- `update_game_window()` routes to `update_screen_window()` for gameplay or `update_interface_display()` for menus
- Fades and screen transitions managed via `interface_fade_in_progress` flag and `animated_color_table`

**Initialization Sequence:**
1. `initialize_application()` ΓåÆ `initialize_game_state()` ΓåÆ sets initial state
2. Main loop calls `idle_game_state()` / `update_interface_display()` until gameplay
3. User selects "New Game" ΓåÆ `begin_game()` ΓåÆ `start_game()` ΓåÆ `_game_in_progress`

**Game End Sequence:**
- `set_game_state(_quit_game)` or `_close_game` ΓåÆ deferred to `idle_game_state()`
- `finish_game()` ΓåÆ cleanup, fade music, unload resources, optionally show stats or menu

## External Dependencies
- **Core Game:** `map.h` (world data), `player.h` (player state), `network.h` (multiplayer), `shell.h` (app lifecycle)
- **Rendering:** `screen_drawing.h` (UI drawing), `render.h` / `OGL_Render.h` / `OGL_Blitter.h` (graphics)
- **Audio:** `SoundManager.h`, `Music.h`, `OpenALManager.h`
- **Resources:** `images.h`, `Movie.h`, `FileHandler.h`, `Plugins.h`
- **Scripting:** `lua_script.h`, `lua_hud_script.h`, `XML_LevelScript.h`
- **Video:** `pl_mpeg.h`, `libyuv/` (MPEG and color conversion)
- **UI/Dialogs:** `sdl_dialogs.h`, `sdl_widgets.h`, `network_dialog_widgets_sdl.h`
- **Utilities:** `cseries.h` (cross-platform macros), `Statistics.h`, `QuickSave.h`, `motion_sensor.h`
- **Conditional:** `steamshim_child.h` (Steam support, ifdef `HAVE_STEAM`)
