# Source_Files/Files/preprocess_map_sdl.cpp

## File Purpose
SDL implementation for save game routines and default resource file discovery. Handles locating essential game data files (maps, shapes, sounds) in the search path, managing game save/load operations, and verifying default resources exist at startup.

## Core Responsibilities
- Locate default game data files (map, physics, shapes, sounds, music, theme) via data search path
- Verify essential default resources exist (`have_default_files()`)
- Manage game save operations with user feedback
- Present save game selection dialogs for loading
- Support post-processing hooks for save file metadata (currently stubbed for macOS compatibility)

## Key Types / Data Structures
None (uses external types: `FileSpecifier`, `DirectorySpecifier` from FileHandler.h).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | global (external) | Directories to search for game data files; defined in shell_sdl.cpp |

## Key Functions / Methods

### get_default_spec (string overload)
- **Signature:** `static bool get_default_spec(FileSpecifier &file, const string &name)`
- **Purpose:** Search `data_search_path` for a file by name; set result into output parameter.
- **Inputs:** `file` (out), `name` (relative file path to search for)
- **Outputs/Return:** `true` if file found and `Exists()`, `false` otherwise
- **Side effects:** Modifies `file` parameter if found
- **Calls:** `FileSpecifier` constructor, `Exists()`, vector iteration
- **Notes:** Iterates through all directories until match found; early exit on success

### get_default_spec (int typecode overload)
- **Signature:** `static bool get_default_spec(FileSpecifier &file, int type)`
- **Purpose:** Convenience overload; fetch filename string resource then call string variant.
- **Inputs:** `file` (out), `type` (string resource ID from strFILENAMES enum)
- **Outputs/Return:** `true` if file found, `false` otherwise
- **Calls:** `getcstr()` (cseries), string overload of `get_default_spec()`
- **Notes:** Bridges string-based lookup with resource-based filenames

### have_default_files
- **Signature:** `bool have_default_files(void)`
- **Purpose:** Verify that essential game files exist (map, shapes, and either images or external resources).
- **Inputs:** None
- **Outputs/Return:** `true` if all required files found
- **Side effects:** None
- **Calls:** `get_default_spec()` (multiple)
- **Notes:** Startup check; logical AND of map, shapes, and (images OR external_resources)

### get_default_external_resources_spec
- **Signature:** `void get_default_external_resources_spec(FileSpecifier& file)`
- **Purpose:** Locate external resources file (optional).
- **Inputs:** `file` (out)
- **Outputs/Return:** None (modifies `file` in-place)
- **Side effects:** Silent failure if not found
- **Calls:** `get_default_spec()`
- **Notes:** No error if missing; used for optional extension content

### get_default_map_spec
- **Signature:** `void get_default_map_spec(FileSpecifier &file)`
- **Purpose:** Locate the default map file; alert on failure.
- **Inputs:** `file` (out)
- **Outputs/Return:** None
- **Side effects:** Calls `alert_bad_extra_file()` if not found
- **Calls:** `get_default_spec()`, `alert_bad_extra_file()`
- **Notes:** Critical resource; alerts user on failure

### get_default_physics_spec
- **Signature:** `void get_default_physics_spec(FileSpecifier &file)`
- **Purpose:** Locate physics model file (optional).
- **Inputs:** `file` (out)
- **Outputs/Return:** None
- **Side effects:** Silent failure
- **Calls:** `get_default_spec()`
- **Notes:** Non-critical; failures ignored

### get_default_shapes_spec
- **Signature:** `void get_default_shapes_spec(FileSpecifier &file)`
- **Purpose:** Locate shapes/graphics file; alert on failure.
- **Inputs:** `file` (out)
- **Outputs/Return:** None
- **Side effects:** Calls `alert_bad_extra_file()` if not found
- **Calls:** `get_default_spec()`, `alert_bad_extra_file()`
- **Notes:** Critical resource

### get_default_sounds_spec
- **Signature:** `void get_default_sounds_spec(FileSpecifier &file)`
- **Purpose:** Locate sounds file (optional).
- **Inputs:** `file` (out)
- **Outputs/Return:** None
- **Side effects:** Silent failure
- **Calls:** `get_default_spec()`
- **Notes:** Non-critical

### get_default_music_spec
- **Signature:** `bool get_default_music_spec(FileSpecifier &file)`
- **Purpose:** Locate music file.
- **Inputs:** `file` (out)
- **Outputs/Return:** `true` if found, `false` otherwise
- **Calls:** `get_default_spec()`
- **Notes:** Return value allows caller to decide if absence is fatal

### get_default_theme_spec
- **Signature:** `bool get_default_theme_spec(FileSpecifier &file)`
- **Purpose:** Locate theme file in "Themes" subdirectory.
- **Inputs:** `file` (out)
- **Outputs/Return:** `true` if found
- **Calls:** `get_default_spec()`, `getcstr()` (indirect via temporary)
- **Notes:** Constructs path as "Themes/<theme_name>"

### choose_saved_game_to_load
- **Signature:** `bool choose_saved_game_to_load(FileSpecifier &saved_game)`
- **Purpose:** Present dialog for user to select a saved game file.
- **Inputs:** `saved_game` (out)
- **Outputs/Return:** `true` if user selected and confirmed
- **Calls:** `load_quick_save_dialog()` (QuickSave.h)
- **Notes:** Wrapper around quick-save dialog; delegates UI to QuickSave module

### save_game
- **Signature:** `bool save_game(void)`
- **Purpose:** Save current game state and provide user feedback.
- **Inputs:** None
- **Outputs/Return:** `true` on success, `false` on failure
- **Side effects:** Creates save file; calls `screen_printf()` to display status message
- **Calls:** `create_quick_save()` (QuickSave.h), `screen_printf()` (shell.h)
- **Notes:** Prints "Game saved" or "Save failed" to screen; delegates to QuickSave module

### add_finishing_touches_to_save_file
- **Signature:** `void add_finishing_touches_to_save_file(FileSpecifier &file)`
- **Purpose:** Post-process saved game file (currently stubbed).
- **Inputs:** `file` (in/out reference to save file)
- **Outputs/Return:** None
- **Side effects:** None (function body is empty)
- **Calls:** None
- **Notes:** Placeholder for future macOS resource-fork handling (thumbnail, level name); currently unused

## Control Flow Notes
- Part of game initialization/shutdown cycle
- `have_default_files()` typically called at startup to verify resources
- `get_default_*_spec()` functions called during engine init to populate resource paths
- `save_game()` and `choose_saved_game_to_load()` called during gameplay
- All I/O operations delegate to `FileSpecifier` abstraction (SDL-based path handling)
- Error handling uses application-wide alert system (`alert_bad_extra_file()`)

## External Dependencies
- **Includes:** `cseries.h` (core macros, types), `FileHandler.h` (FileSpecifier), `world.h` (coordinate types), `map.h` (map data), `shell.h` (application shell), `interface.h` (resource strings, UI), `game_wad.h` (save format), `game_errors.h` (error system), `QuickSave.h` (save dialogs/operations)
- **External symbols:** `data_search_path` (from shell_sdl.cpp), `getcstr()`, `alert_bad_extra_file()`, `load_quick_save_dialog()`, `create_quick_save()`, `screen_printf()` (defined elsewhere)
- **Standard library:** `<vector>`, `<string>`
