# Source_Files/Misc/preferences.cpp

## File Purpose
Preferences handling system for Aleph One game engine. Manages initialization, UI dialogs, parsing, validation, and persistence of all user preferences across graphics, networking, input controls, sound, player data, and environment settings.

## Core Responsibilities
- Initialize default preferences for all subsystems on startup
- Provide interactive preference dialogs organized by category (player, graphics, network, sound, controls, environment)
- Parse preference data from XML-based configuration files (MML/InfoTree format)
- Validate loaded preferences for consistency and safety
- Apply environment preferences to the running game
- Handle password obfuscation for metaserver credentials
- Manage plugin enable/disable state in preferences
- Support legacy preference format migration from old versions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `graphics_preferences_data` | struct | Screen mode, resolution, OpenGL settings, video export quality |
| `network_preferences_data` | struct | Network game type, difficulty, time/kill limits, metaserver credentials, game port |
| `player_preferences_data` | struct | Player name, color, team, crosshair/chase-cam settings, solo gameplay profile |
| `input_preferences_data` | struct | Input device, key bindings, mouse sensitivity, controller deadzone, modifiers |
| `environment_preferences_data` | struct | Map/physics/shapes/sounds file paths and checksums, plugin lists, Lua script settings |
| `CrosshairPref`, `ColorComponentPref`, `OpacityPref` | class | Bindable adapters converting between preference types and UI widget values |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `graphics_preferences` | `graphics_preferences_data*` | global | Current graphics preferences (allocated at init) |
| `network_preferences` | `network_preferences_data*` | global | Current network preferences (allocated at init) |
| `player_preferences` | `player_preferences_data*` | global | Current player preferences (allocated at init) |
| `input_preferences` | `input_preferences_data*` | global | Current input preferences (allocated at init) |
| `sound_preferences` | `SoundManager::Parameters*` | global | Current sound preferences (allocated at init) |
| `environment_preferences` | `environment_preferences_data*` | global | Current environment preferences (allocated at init) |
| `PrefsInited` | bool (static) | file-static | Flag indicating whether preferences have been initialized |
| `orphan_disabled_plugins`, `orphan_enabled_plugins` | vectors of paths | file-static | Plugins referenced in prefs but not found on disk |
| `sPasswordMask` | const char[] (static) | file-static | String used for XOR obfuscation of metaserver password |

## Key Functions / Methods

### handle_preferences
- **Signature:** `void handle_preferences(void)`
- **Purpose:** Main entry point; displays master preferences dialog with buttons for each preference category
- **Inputs:** None (reads UI interaction)
- **Outputs/Return:** None; modifies global preference structures
- **Side effects:** Saves current preferences before dialog, displays screen UI, may call `write_preferences()`
- **Calls:** `write_preferences()`, `player_dialog()`, `online_dialog()`, `graphics_dialog()`, `sound_dialog()`, `controls_dialog()`, `environment_dialog()`, `plugins_dialog()`, dialog widget constructors
- **Notes:** Uses vertical placer widget layout; calls `clear_screen()` and `display_main_menu()` around dialog execution

### player_dialog
- **Signature:** `static void player_dialog(void *arg)`
- **Purpose:** Preferences dialog for player appearance and gameplay options (name, color, team, difficulty, crosshair settings, solo profile)
- **Inputs:** Dialog parent pointer (cast from void)
- **Outputs/Return:** None; modifies `player_preferences` on accept
- **Side effects:** Calls `write_preferences()` if changes detected; shows solo gameplay profile options if scenario allows
- **Calls:** Dialog widget constructors (`w_text_entry`, `w_select`, `w_player_color`, `w_toggle`), `crosshair_dialog()`, preference comparison/assignment
- **Notes:** Includes nested crosshair settings button; handles name entry with Mac Roman character support; uses legacy field mapping for solo_profile

### online_dialog
- **Signature:** `static void online_dialog(void *arg)`
- **Purpose:** Preferences dialog for metaserver account, pregame lobby, and stats settings
- **Inputs:** Dialog parent pointer (cast from void)
- **Outputs/Return:** None; modifies `network_preferences` and `player_preferences` on accept
- **Side effects:** Calls `write_preferences()`, `signup_dialog()`, `proc_account_link()`; manages HTTP login and account creation
- **Calls:** Tab widget system, `signup_dialog()`, `proc_account_link()`, HTTP client (via `proc_account_link`)
- **Notes:** Three tabs (account, pregame lobby, stats); password handled specially with encryption; custom chat colors with dependent widget enable/disable

### crosshair_dialog
- **Signature:** `static void crosshair_dialog(void *arg)`
- **Purpose:** Preferences dialog for detailed crosshair customization (shape, thickness, color, opacity, position)
- **Inputs:** Dialog parent pointer (cast from void)
- **Outputs/Return:** None; modifies `player_preferences->Crosshairs` on accept
- **Side effects:** Calls `write_preferences()` if changes accepted; uses `BinderSet` for bidirectional widget-data synchronization
- **Calls:** Table placer, slider widgets, `w_crosshair_display`, `BinderSet` migrate operations
- **Notes:** Uses custom adapter classes (`CrosshairPref`, `ColorComponentPref`, `OpacityPref`) for type conversion; OpenGL-only features noted in UI

### default_graphics_preferences
- **Signature:** `static void default_graphics_preferences(graphics_preferences_data *preferences)`
- **Purpose:** Initialize graphics preferences to sensible defaults
- **Inputs:** Preferences structure pointer
- **Outputs/Return:** None; fills structure with default values
- **Side effects:** None
- **Calls:** None
- **Notes:** Sets 640├ù480 windowed, 16-bit color, OpenGL enabled, 30 FPS target, medium ephemera quality

### default_network_preferences
- **Signature:** `static void default_network_preferences(network_preferences_data *preferences)`
- **Purpose:** Initialize network preferences to sensible defaults
- **Inputs:** Preferences structure pointer
- **Outputs/Return:** None; fills structure with default values
- **Side effects:** Calls `DefaultStarPreferences()` if networking enabled
- **Calls:** `DefaultStarPreferences()`, `get_interface_color()`
- **Notes:** Defaults to ethernet, kill-monsters game type, 10-minute time limit, guest metaserver login

### default_player_preferences
- **Signature:** `static void default_player_preferences(player_preferences_data *preferences)`
- **Purpose:** Initialize player preferences to sensible defaults
- **Inputs:** Preferences structure pointer
- **Outputs/Return:** None; fills structure with default values
- **Side effects:** None
- **Calls:** `get_name_from_system()`
- **Notes:** Gets username from OS, sets default crosshair and chase-cam parameters

### default_input_preferences
- **Signature:** `static void default_input_preferences(input_preferences_data *preferences)`
- **Purpose:** Initialize input preferences to sensible defaults
- **Inputs:** Preferences structure pointer
- **Outputs/Return:** None; fills structure with default values
- **Side effects:** None
- **Calls:** None
- **Notes:** Mouse yaw/pitch control, 1/4 horizontal/vertical sensitivity, no acceleration, raw mouse input enabled

### default_environment_preferences
- **Signature:** `static void default_environment_preferences(environment_preferences_data *preferences)`
- **Purpose:** Initialize environment preferences to sensible defaults
- **Inputs:** Preferences structure pointer
- **Outputs/Return:** None; fills structure with default values
- **Side effects:** None
- **Calls:** `get_default_map_spec()`, `get_default_physics_spec()`, `get_default_shapes_spec()`, `get_default_sounds_spec()`, `get_default_external_resources_spec()`, `read_wad_file_checksum()`
- **Notes:** Caches checksums and modification dates of default files; sets up QuickSaves limits based on Steam availability

### parse_graphics_preferences, parse_player_preferences, parse_input_preferences, parse_sound_preferences, parse_network_preferences, parse_environment_preferences
- **Signature:** `void parse_*_preferences(InfoTree root, std::string version)`
- **Purpose:** Parse preference data from XML tree (MML format) into preference structures
- **Inputs:** InfoTree root node, version string for backward compatibility
- **Outputs/Return:** None; updates corresponding global preference structure
- **Side effects:** Reads attribute values and child nodes from XML; applies format conversions for legacy versions
- **Calls:** InfoTree methods (`read_attr()`, `read_cstr()`, `read_path()`, `children_named()`, `read_color()`), version comparison
- **Notes:** Heavy use of version checks for backward compatibility; handles old key binding format translation via `translate_old_key()`

### load_environment_from_preferences
- **Signature:** `void load_environment_from_preferences(void)`
- **Purpose:** Apply current environment preferences to the game (load maps, physics, shapes, sounds)
- **Inputs:** None; reads global `environment_preferences`
- **Outputs/Return:** None; modifies game world state
- **Side effects:** File I/O; calls game engine functions to load/set files
- **Calls:** `set_map_file()`, `import_definition_structures()`, `set_physics_file()`, `open_shapes_file()`, `SoundManager::OpenSoundFile()`, `set_external_resources_file()`, `find_wad_file_that_has_checksum()`, `find_file_with_modification_date()`
- **Notes:** Tries to match by path first, then by checksum/modification date as fallback

### validate_graphics_preferences, validate_network_preferences, validate_player_preferences, validate_input_preferences, validate_environment_preferences
- **Signature:** `static bool validate_*_preferences(*_preferences_data *preferences)`
- **Purpose:** Validate and correct preferences loaded from file
- **Inputs:** Preferences structure pointer
- **Outputs/Return:** true if changes were made, false otherwise
- **Side effects:** Modifies preference fields to clamp/correct invalid values
- **Calls:** None (or helper functions like `ethernet_active()`)
- **Notes:** Ensures bool fields are actually bool, clamps numeric ranges, forces OpenGL compatibility (e.g., 16-bit minimum)

### get_name_from_system
- **Signature:** `static std::string get_name_from_system()`
- **Purpose:** Retrieve OS username for default player name
- **Inputs:** None
- **Outputs/Return:** Username string (or "Bob User" fallback)
- **Side effects:** None
- **Calls:** `getlogin()` (Unix), `GetUserNameW()` (Windows), `wide_to_utf8()` (Windows)
- **Notes:** Platform-conditional implementation; returns UTF-8 string on all platforms

### ethernet_active
- **Signature:** `static bool ethernet_active(void)`
- **Purpose:** Check if network connectivity is available
- **Inputs:** None
- **Outputs/Return:** true (always returns true in current code)
- **Side effects:** None
- **Calls:** None
- **Notes:** Stub function; historically checked for network availability but now assumes always available

### translate_old_key
- **Signature:** `SDL_Scancode translate_old_key(int code)`
- **Purpose:** Convert old-format key codes to SDL2 scancodes for backward compatibility
- **Inputs:** Integer key code from legacy preference format
- **Outputs/Return:** Corresponding SDL_Scancode (or SDL_SCANCODE_UNKNOWN)
- **Side effects:** None
- **Calls:** `SDL_GetScancodeFromKey()`
- **Notes:** Large lookup table for classic ASCII and special keys; handles mouse buttons and joystick buttons specially

### get_hotkey_binding
- **Signature:** `const char* get_hotkey_binding(int hotkey, int type)`
- **Purpose:** Get human-readable key name for a hotkey at specified event type
- **Inputs:** Hotkey index (1-based), event type
- **Outputs/Return:** Key name string (or empty string if not bound)
- **Side effects:** None
- **Calls:** `GetSDLKeyName()`, `w_key::event_type_for_key()`
- **Notes:** Iterates hotkey bindings set to find matching event type

## Control Flow Notes
- **Initialization:** `initialize_preferences()` (defined elsewhere) allocates all preference structures and sets defaults
- **Game Frame:** Preferences are read from disk once at startup; updates happen only through explicit user interaction with preference dialogs
- **Preference Dialogs:** Called from shell/interface code when user requests preferences; `handle_preferences()` creates a master dialog menu, user selects category, category-specific dialog runs, changes are validated and written to disk
- **Shutdown:** Preferences are persisted to disk at exit (via `write_preferences()` called from shell shutdown)

## External Dependencies
- **Widget System:** Dialog, vertical_placer, horizontal_placer, table_placer, w_button, w_text_entry, w_select, w_toggle, w_slider, w_color_picker, w_player_color, w_crosshair_display, w_enabling_toggle, w_hyperlink (from SDL/dialog headers)
- **InfoTree:** XML preference parsing (from InfoTree.h)
- **FileHandler:** File I/O operations (FileSpecifier, DirectorySpecifier, OpenedFile)
- **Network:** Network protocol handling (StarGameProtocol::ParsePreferencesTree), HTTP client (HTTPClient)
- **Sound:** SoundManager for sound parameters
- **Game World:** Map loading (set_map_file, set_physics_file, import_definition_structures), shape/sound file management
- **Plugin System:** Plugins::instance() for enable/disable operations
- **Shell/Interface:** screen_mode_data, interface color functions, dialog callbacks
- **SDL2:** SDL_Color, SDL_Scancode, key name functions
- **Standard Library:** std::string, std::vector, std::map, std::set, Boost (hex algorithm, filesystem paths)
- **Platform-Specific:** Wide character conversion (wide_to_utf8), Windows API (GetUserNameW, UNLEN)
