# Source_Files/Misc/Console.cpp

## File Purpose
Implements an interactive console system for Aleph One with text input, command execution, command history, macros, and multiplayer kill-message reporting (carnage messages). The console is a singleton that handles both user input processing and configuration via MML.

## Core Responsibilities
- Manage console input state (activate/deactivate, buffer, cursor positioning)
- Process text events and editing commands (insert, delete, backspace, move cursor)
- Parse and execute console commands with macro expansion
- Maintain and navigate command history (up/down arrow)
- Register and execute registered command functions
- Configure and report kill messages for networked multiplayer games
- Load console configuration (macros, carnage messages) from MML files
- Support level saving via console command

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `CommandParser` | class | Base class managing command registration and execution; maps command strings to callback functions |
| `Console` | class | Singleton extending CommandParser; manages full console state including input, history, macros, carnage messages |
| `save_level` | struct (functor) | Command handler for saving the current game level |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `last_level` | `std::string` | static (file) | Stores the filename of the last saved level for quick re-save |
| `game_is_networked` | `bool` | extern | Flag indicating if the game is in networked multiplayer mode |

## Key Functions / Methods

### Console::instance()
- **Signature:** `static Console* instance()`
- **Purpose:** Lazy-initialized singleton accessor.
- **Inputs:** None.
- **Outputs/Return:** Pointer to the single Console instance.
- **Side effects:** Creates the Console on first call.
- **Calls:** Console constructor (private).
- **Notes:** Thread-unsafe; assumes single-threaded initialization.

### Console::activate_input()
- **Signature:** `void activate_input(std::function<void(const std::string&)> callback, const std::string& prompt)`
- **Purpose:** Start accepting console input; sets up SDL text input and displays the prompt.
- **Inputs:** `callback` (function to invoke on enter or abort with the command/empty string), `prompt` (display label).
- **Outputs/Return:** None.
- **Side effects:** Sets `m_active=true`, clears buffers, calls `SDL_StartTextInput()`, sets `m_cursor_position=0`.
- **Calls:** SDL_StartTextInput(), SDL_FlushEvent().
- **Notes:** Asserts that input is not already active.

### Console::deactivate_input()
- **Signature:** `void deactivate_input()`
- **Purpose:** Stop accepting console input without invoking the callback (unlike `abort()`).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Clears buffers, nulls callback, sets `m_active=false`, calls `SDL_StopTextInput()`.
- **Calls:** SDL_StopTextInput().
- **Notes:** Differs from `abort()` in that no callback is invoked.

### Console::enter()
- **Signature:** `void enter()`
- **Purpose:** Process the entered command: store in history, expand macros, execute as command or invoke callback.
- **Inputs:** None (uses `m_buffer`).
- **Outputs/Return:** None.
- **Side effects:** Adds command to `m_prev_commands` if not duplicate; resets `m_command_iter`; clears `m_buffer` and `m_displayBuffer`; sets `m_active=false`; calls `SDL_StopTextInput()`.
- **Calls:** `parse_and_execute()`, `m_callback()`, SDL_StopTextInput().
- **Notes:** Commands starting with `.` are treated as console commands; otherwise callback is invoked. Macros are expanded before command execution.

### Console::report_kill()
- **Signature:** `void report_kill(int16 player_index, int16 aggressor_player_index, int16 projectile_index)`
- **Purpose:** Display a kill message in networked multiplayer when a player kills another (or suicides).
- **Inputs:** `player_index` (killed player), `aggressor_player_index` (killer), `projectile_index` (weapon used).
- **Outputs/Return:** None.
- **Side effects:** Calls `screen_printf()` to display formatted kill message.
- **Calls:** `NetAllowCarnageMessages()`, `get_projectile_data()`, `get_player_data()`, `screen_printf()`, `replace_first()`.
- **Notes:** Returns early if not networked, carnage messages disabled, or no messages configured. Replaces `%player%` and `%aggressor%` tokens in message template; order-dependent to avoid double replacement.

### CommandParser::parse_and_execute()
- **Signature:** `void parse_and_execute(const std::string& command_string)`
- **Purpose:** Split command string and invoke registered command handler.
- **Inputs:** `command_string` (command with optional arguments).
- **Outputs/Return:** None.
- **Side effects:** Looks up command in `m_commands` map and invokes the associated function with remainder string.
- **Calls:** `split()`, `lowercase()`, map lookup and function invocation.
- **Notes:** Silent no-op if command not found.

### Console::textEvent()
- **Signature:** `void textEvent(const SDL_Event &e)`
- **Purpose:** Handle SDL text input event; insert character(s) into buffer at cursor.
- **Inputs:** SDL_Event with text data.
- **Outputs/Return:** None.
- **Side effects:** Inserts UTF-8 input (converted to Mac Roman) into `m_buffer` and `m_displayBuffer` at cursor position; advances cursor.
- **Calls:** `utf8_to_mac_roman()`.
- **Notes:** Handles multi-byte UTF-8 sequences.

### Console::register_macro()
- **Signature:** `void register_macro(std::string input, std::string output)`
- **Purpose:** Register a macro (text expansion).
- **Inputs:** `input` (macro name), `output` (replacement text).
- **Outputs/Return:** None.
- **Side effects:** Adds entry to `m_macros` map (lowercased key).
- **Calls:** `lowercase()`.
- **Notes:** Macros are expanded during `enter()` when command starts with `.`.

### Console::set_carnage_message()
- **Signature:** `void set_carnage_message(int16 projectile_type, const std::string& on_kill, const std::string& on_suicide="")`
- **Purpose:** Configure a kill message template for a projectile type.
- **Inputs:** `projectile_type` (weapon index), `on_kill` (message when other player killed), `on_suicide` (message when self-inflicted kill).
- **Outputs/Return:** None.
- **Side effects:** Sets `m_carnage_messages_exist=true`; stores messages in `m_carnage_messages` vector.
- **Calls:** None.
- **Notes:** Supports `%player%` and `%aggressor%` token substitution.

### parse_mml_console()
- **Signature:** `void parse_mml_console(const InfoTree& root)`
- **Purpose:** Load console configuration (use_lua_console flag, macros, carnage messages) from MML.
- **Inputs:** `root` (InfoTree containing console element and children).
- **Outputs/Return:** None.
- **Side effects:** Calls `Console::instance()` and modifies its state via `use_lua_console()`, `register_macro()`, `set_carnage_message()`.
- **Calls:** Console instance methods, InfoTree readers.
- **Notes:** Iterates over `<macro>` and `<carnage_message>` child elements; skips malformed entries.

## Control Flow Notes
**Console activation flow:** Game code calls `activate_input()` ΓåÆ console becomes active and receives SDL text events ΓåÆ each keystroke handled by edit functions (`textEvent()`, `backspace()`, etc.) ΓåÆ when user presses Enter, `enter()` is called ΓåÆ macro expansion, then command execution or callback invocation ΓåÆ console deactivated and input stops.

**Initialization:** Console is loaded from MML via `parse_mml_console()` during engine startup, configuring macros and kill messages. The level-save command is registered in the constructor via `register_save_commands()`.

## External Dependencies
- **SDL2:** `SDL_Event`, `SDL_StartTextInput()`, `SDL_StopTextInput()`, `SDL_FlushEvent()`
- **std library:** `<functional>`, `<string>`, `<map>`, `std::bind()`, `std::transform()`
- **boost:** `boost::algorithm::ends_with()` for filename extension checking
- **cseries.h:** Platform abstractions, types
- **Logging.h:** `logAnomaly()` macro
- **network.h:** `game_is_networked`, `NetAllowCarnageMessages()`, `NetAllowSavingLevel()`
- **player.h:** `get_player_data()`
- **projectiles.h:** `projectile_data`, `get_projectile_data()`, `NUMBER_OF_PROJECTILE_TYPES`
- **FileHandler.h:** `FileSpecifier`
- **game_wad.h:** `export_level()`
- **InfoTree.h:** MML parsing
