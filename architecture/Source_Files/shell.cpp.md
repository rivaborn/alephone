# Source_Files/shell.cpp

## File Purpose
Main entry point and control flow for the Aleph One game engine. Orchestrates SDL initialization, event dispatching, the main game loop, input handling (keyboard, mouse, joystick, controller), and application lifecycle management.

## Core Responsibilities
- **Application lifecycle**: Initialize SDL, load resources, game state management, clean shutdown
- **Event loop**: Main loop that polls SDL events, executes timer tasks, manages game state transitions
- **Input handling**: Keyboard shortcuts (F-keys for graphics, console, etc.), cheat key detection, game/menu navigation
- **Resource initialization**: Data directories, MML scripts, preferences, collections, sound/music systems
- **File handling**: Document opening (scenario, savegame, film, physics files), screenshot capture, symbolic path expansion/contraction
- **Directory management**: Construct data search path, preferences, saves, recordings, caches with platform-specific paths

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `screen_mode_data` (shell.h) | struct | Graphics mode settings (resolution, fullscreen, HUD, gamma, etc.) |
| `view_data` (render.h) | struct | Rendering view parameters (FOV, screen dimensions, camera position, effects) |
| `entry_point` (map.h) | struct | Level start location and metadata |
| `game_data` (map.h) | struct | Game type, options, difficulty, kill limits |
| `dynamic_data` (map.h) | struct | Runtime game state (tick count, random seed, counts of entities) |
| `static_data` (map.h) | struct | Map-level metadata (environment, physics model, music, mission flags) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | global | Colon-separated list of directories to search for game data |
| `local_data_dir` | `DirectorySpecifier` | global | Per-user local data directory |
| `default_data_dir` | `DirectorySpecifier` | global | Default scenario/game data directory |
| `bundle_data_dir` | `DirectorySpecifier` | global | macOS app bundle data (if applicable) |
| `preferences_dir` | `DirectorySpecifier` | global | User preferences storage |
| `saved_games_dir` | `DirectorySpecifier` | global | Saved game files location |
| `quick_saves_dir` | `DirectorySpecifier` | global | Auto-named quicksaves |
| `image_cache_dir` | `DirectorySpecifier` | global | Cached image textures |
| `recordings_dir` | `DirectorySpecifier` | global | Film/replay recordings |
| `screenshots_dir` | `DirectorySpecifier` | global | Screenshots from dump_screen() |
| `log_dir` | `DirectorySpecifier` | global | Aleph One log file |
| `subscribed_workshop_items` (HAVE_STEAM) | `vector<item>` | global | Steam workshop mod list |
| `steam_game_info` (HAVE_STEAM) | `steam_game_information` | global | Steam game metadata |
| `vidmasterStringSetID` | `short` | static | String set ID for custom vidmaster oath (MML-configurable) |
| `vidmasterLevelOffset` | `short` | static | Starting level offset for vidmaster mode |

## Key Functions / Methods

### initialize_application
- **Signature:** `void initialize_application(void)`
- **Purpose:** One-time initialization of the entire game engine on startup
- **Inputs:** None (reads `shell_options` global)
- **Outputs/Return:** None; sets up all global state
- **Side effects:** 
  - Initializes SDL with video, audio, joystick subsystems (conditional)
  - Loads MML scripts, fonts, collections, preferences
  - Initializes physics, sound manager, music handler, rendering, terminal manager, shape handler, fades, images, dialogs
  - Creates/discovers scenario directories; may prompt user if no scenario found
  - Loads Steam workshop items (if HAVE_STEAM)
- **Calls:** `initialize_joystick()`, `initialize_resources()`, `initialize_fonts()`, `load_film_profile()`, `LoadBaseMMLScripts()`, `have_default_files()`, `alert_choose_scenario()`, `Plugins::enumerate()`, `initialize_preferences()`, `WadImageCache::initialize_cache()`, `Plugins::load_mml()`, `HTTPClient::Init()`, `mytm_initialize()`, `SoundManager::Initialize()`, `initialize_marathon_music_handler()`, `initialize_keyboard_controller()`, `initialize_gamma()`, `alephone::Screen::instance()->Initialize()`, `initialize_marathon()`, `initialize_screen_drawing()`, `initialize_dialogs()`, `initialize_terminal_manager()`, `initialize_shape_handler()`, `initialize_fades()`, `initialize_images_manager()`, `load_environment_from_preferences()`, `initialize_game_state()`
- **Notes:** Loads command-line options from `shell_options` struct. Handles scenario selection via UI or command-line. Validates presence of critical files (strERRORS, strFILENAMES). Supports platform-specific behaviors (Mac bundle, Steam integration, Windows debug mode).

### shutdown_application
- **Signature:** `void shutdown_application(void)`
- **Purpose:** Clean shutdown of engine and SDL
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Saves image cache, shuts down dialogs, image libraries, font support, SDL, Steam shim
- **Calls:** `WadImageCache::instance()->save_cache()`, `shutdown_dialogs()`, `IMG_Quit()`, `TTF_Quit()`, `SDL_Quit()`, `STEAMSHIM_deinit()`
- **Notes:** Inverse of `initialize_application()`.

### main_event_loop
- **Signature:** `void main_event_loop(void)`
- **Purpose:** The core game loop; repeatedly polls events, updates game state, and idles until quit
- **Inputs:** None (reads global game state)
- **Outputs/Return:** None; runs until `get_game_state() == _quit_game`
- **Side effects:** 
  - Dispatches SDL events via `process_event()`
  - Executes timer tasks and idles game state each frame
  - Manages FPS capping (target 60 Hz during gameplay or 30 Hz in menus)
  - Pauses on Steam overlay activation (if HAVE_STEAM)
- **Calls:** `get_game_state()`, `machine_tick_count()`, `get_fps_target()`, `get_keyboard_controller_status()`, `Console::instance()->input_active()`, `interface_fade_finished()`, `global_idle_proc()`, `SDL_WaitEventTimeout()`, `process_event()`, `SDL_PollEvent()`, `STEAMSHIM_pump()`, `pause_game()`, `execute_timer_tasks()`, `idle_game_state()`, `sleep_for_machine_ticks()`, `update_game_window()`
- **Notes:** Conditional polling based on game state; keyboard-controlled gameplay gets full 60 Hz, menus/cutscenes yield time. FPS target can be overridden by preferences.

### process_event
- **Signature:** `static void process_event(const SDL_Event &event)`
- **Purpose:** Dispatch SDL events to appropriate handlers (mouse, keyboard, controller, window, etc.)
- **Inputs:** `event` - SDL event from event queue
- **Outputs/Return:** None; updates game state via side effects
- **Side effects:** 
  - Mouse: calls `mouse_moved()`, `mouse_scroll()`, or `resume_game()` if keyboard-controlled
  - Controller: calls `joystick_*()` functions, may resume game or trigger menu actions
  - Keyboard: calls `process_game_key()` for all key presses
  - Text input: forwards to `Console::instance()->textEvent()`
  - Quit: triggers quit menu or sets quit state
  - Window: handles focus loss (pause if active gameplay), focus gain (macOS Mojave workaround)
  - Text input: forwards raw text to console if console input is active
- **Calls:** `mouse_moved()`, `mouse_scroll()`, `resume_game()`, `get_keyboard_controller_status()`, `joystick_button_pressed()`, `joystick_axis_moved()`, `joystick_added()`, `joystick_removed()`, `pause_game()`, `process_game_key()`, `Console::instance()->textEvent()`, `do_menu_item_command()`, `set_game_state()`, `set_game_focus_lost()`, `set_game_focus_gained()`
- **Notes:** Synthesizes SDL_KEYDOWN events for mouse buttons and joystick buttons (using custom scancode offsets). Handles platform-specific fullscreen issues on macOS Mojave.

### process_game_key
- **Signature:** `static void process_game_key(const SDL_Event &event)`
- **Purpose:** Route keyboard input to game or menu handlers based on current game state
- **Inputs:** `event` - SDL key event
- **Outputs/Return:** None
- **Side effects:** Delegates to `handle_game_key()` or menu-specific handlers; triggers menu commands or game features (pause, save, quit, fullscreen toggle)
- **Calls:** `interface_fade_finished()`, `stop_interface_fade()`, `force_game_state_change()`, `do_menu_item_command()`, `toggle_fullscreen()`, `handle_game_key()`, `process_main_menu_highlight_*()`, `draw_menu_button_for_command()`
- **Notes:** Detects Alt+/Cmd+ modifiers for menu shortcuts (P=Pause, S=Save, R=Revert, Q=Quit). Main menu has keyboard shortcuts (N=New, O=Load, etc.) and joystick support for D-pad and A button.

### handle_game_key
- **Signature:** `static void handle_game_key(const SDL_Event &event)`
- **Purpose:** Process in-game hotkeys (F-keys for HUD, console, gamma, camera, etc.; console input shortcuts)
- **Inputs:** `event` - SDL key event
- **Outputs/Return:** None
- **Side effects:** 
  - Console input active: handles line editing (return, escape, backspace, arrows, Ctrl+A/B/D/E/F/H/K/N/P/T/U/W)
  - F1/F2: toggle or cycle screen size / HUD
  - F3: toggle resolution (high/low/interlaced) in software mode
  - F4: reset OpenGL textures
  - F5: switch chase cam sides
  - F6: toggle chase cam
  - F7: toggle tunnel vision
  - F8: toggle crosshairs
  - F9: dump screen
  - F10: toggle position display
  - F11/F12: adjust gamma (with Shift on Steam)
  - Volume up/down: adjust master volume
  - Switch view: cycle player list
  - Zoom in/out: map zoom with feedback
  - Inventory left/right: scroll inventory or adjust replay speed
  - Toggle FPS/scores/console: toggle HUD elements
  - Escape: quit or return to menu (context-dependent, requires time in game)
- **Calls:** `Console::instance()->*()`, `PlayInterfaceButtonSound()`, `toggle_fullscreen()`, `do_menu_item_command()`, `zoom_overhead_map_*()`, `ChaseCam_*()`, `SetTunnelVision()`, `scroll_inventory()`, `increment_replay_speed()`, `decrement_replay_speed()`, `dump_screen()`, `change_gamma_level()`, `SoundManager::instance()->AdjustVolume*()`, `walk_player_list()`, `render_screen()`, `ExecuteLuaString()`, `change_screen_mode()`, `OGL_ResetTextures()`, `mouse_scroll()`, `resume_game()`
- **Notes:** Many keys are conditional on game state (some only work during gameplay, some only in replay). Cheat key detection (Shift+Ctrl, varies by platform) gates some menu commands. Player preferences control some behavior (button sounds, crosshairs, HUD scale).

### get_level_number_from_user
- **Signature:** `short get_level_number_from_user(void)`
- **Purpose:** Present a dialog to select a playable level, with optional vidmaster oath text
- **Inputs:** None (reads `get_entry_points()`, `TS_IsPresent()`, `vidmasterStringSetID`)
- **Outputs/Return:** `short` - selected level number, or `NONE` if cancelled
- **Side effects:** Creates and runs a modal dialog; updates game window on dismiss
- **Calls:** `get_entry_points()`, `TS_IsPresent()`, `TS_CountStrings()`, `TS_GetCString()`, `update_game_window()`
- **Notes:** If `vidmasterStringSetID` is set and has strings, displays custom oath text; otherwise uses hardcoded Marathon 1 vidmaster oath. Level offset is configurable via `vidmasterLevelOffset` (MML-settable).

### quit_without_saving
- **Signature:** `bool quit_without_saving(void)`
- **Purpose:** Show confirmation dialog before quitting or cancelling active game
- **Inputs:** None
- **Outputs/Return:** `bool` - true if user confirmed, false if cancelled
- **Side effects:** Creates and runs modal dialog
- **Calls:** (dialog widget constructors and methods)
- **Notes:** Simple two-button (YES/NO) confirmation with hardcoded text.

### handle_open_document
- **Signature:** `bool handle_open_document(const std::string& filename)`
- **Purpose:** Open and load a file dropped on the application (macOS/Windows drag-and-drop)
- **Inputs:** `filename` - path to file
- **Outputs/Return:** `bool` - true if file was handled (loaded and ready), false otherwise
- **Side effects:** Loads file based on type; may set physics, map, shapes, sounds, or start game
- **Calls:** `FileSpecifier::GetType()`, `set_map_file()`, `handle_edit_map()`, `load_and_start_game()`, `handle_open_replay()`, `set_physics_file()`, `open_shapes_file()`, `SoundManager::OpenSoundFile()`
- **Notes:** File type is determined via `FileSpecifier::GetType()`, which checks file typecode. Supports scenarios, savegames, films, physics, shapes, sounds.

### dump_screen
- **Signature:** `void dump_screen(void)`
- **Purpose:** Capture and save current screen to PNG or BMP file
- **Inputs:** None (reads `get_game_state()`, `static_world->level_name`, `MainScreenSurface()`, rendering state)
- **Outputs/Return:** None; writes file to disk
- **Side effects:** 
  - Determines filename (level name + 4-digit counter, or "Screenshot_####")
  - For software renderer: directly saves surface via SDL_SaveBMP or IMG_SavePNG
  - For OpenGL: reads frame buffer, flips vertically (OpenGL is upside-down), saves result
- **Calls:** `get_game_state()`, `to_alnum()`, `MainScreenSurface()`, `SDL_SaveBMP()` or `IMG_SavePNG()`, `MainScreenIsOpenGL()`, `MainScreenPixelWidth()`, `MainScreenPixelHeight()`, `SDL_CreateRGBSurface()`, `malloc()`, `glPixelStorei()`, `glReadPixels()`, `memcpy()`, `SDL_FreeSurface()`
- **Notes:** OpenGL path allocates temporary surfaces and pixel buffers; must handle endianness correctly. Filename is sanitized with `to_alnum()` to remove special characters.

### LoadBaseMMLScripts
- **Signature:** `void LoadBaseMMLScripts(bool load_menu_mml_only)`
- **Purpose:** Load and parse all MML (Maraca Markup Language) scripts from data directories
- **Inputs:** `load_menu_mml_only` - if true, only load menu/UI configuration; if false, load all
- **Outputs/Return:** None (parses into global state)
- **Side effects:** Enumerates "MML" and "Scripts" subdirectories in all search paths; parses each file via `ParseMMLFromFile()`
- **Calls:** `_ParseMMLDirectory()`
- **Notes:** Skips files ending with `~` (backup files) and `.lua` files. Order matters (sorted by `_ParseMMLDirectory`).

## Control Flow Notes

**Init/Shutdown:**
- `initialize_application()` called once on startup; establishes data dirs, loads resources, initializes all subsystems
- `shutdown_application()` called on exit to clean up SDL and save state

**Main Loop:**
- `main_event_loop()` is the event-driven core; runs until game state is `_quit_game`
- Per-frame: poll SDL events, dispatch to handlers, execute timer tasks, idle game state, sleep to cap FPS

**Game State Machine:**
- `get_game_state()` / `set_game_state()` manage transitions between menus, gameplay, cutscenes, networking dialogs
- `process_game_key()` and `process_event()` route input to state-specific handlers
- Key states: `_game_in_progress`, `_display_main_menu`, `_display_intro_screens`, `_quit_game`, `_change_level`, etc.

**Input Handling:**
- SDL events ΓåÆ `process_event()` ΓåÆ state-specific dispatch
- Keyboard/gamepad ΓåÆ `process_game_key()` ΓåÆ `handle_game_key()` (in-game) or menu handlers
- Mouse/controller ΓåÆ direct function calls (e.g., `mouse_moved()`, `joystick_*()`)

## External Dependencies
- **SDL 2:** Video, audio, input, event loop (SDL.h, SDL_image.h, SDL_ttf.h)
- **Game subsystems:** map.h, monsters.h, player.h, render.h, interface.h, SoundManager.h, Music.h, Crosshairs.h, ChaseCam.h, Console.h, Movie.h, Network.h, Lua (lua_script.h)
- **File/resource management:** FileHandler.h, resource_manager.h, game_wad.h, WadImageCache.h, Plugins.h
- **Platform-specific:** shell_options.h, mytm.h, shell_*.h (platform variants), steamshim_child.h (Steam)
- **Graphics/UI:** OGL_Render.h, OGL_Blitter.h, screen.h, screen_drawing.h, sdl_dialogs.h, sdl_widgets.h, interface_menus.h
- **Utilities:** cseries.h, csmacros.h, preferences.h, DefaultStringSets.h, TextStrings.h, alephversion.h, Logging.h, HTTP.h, FilmProfile.h, ScenarioChooser.h
- **Boost:** algorithm/string/predicate.hpp (ends_with)

**Not Defined Here:**
- Game state management, menu rendering, networking, physics simulation, monster/player AIΓÇöall defined in included headers or separate files
