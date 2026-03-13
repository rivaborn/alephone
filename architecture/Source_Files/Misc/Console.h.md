# Source_Files/Misc/Console.h

## File Purpose
Console input and command-execution utilities for Aleph One (Marathon engine). Provides a singleton console with keyboard-driven line editing, command registration/dispatch, command history navigation, text macros, and kill-reporting (carnage) functionality.

## Core Responsibilities
- Register and dispatch console commands with string arguments
- Handle keyboard input (text editing, navigation, deletion, history)
- Manage command history (up/down arrow navigation)
- Support text macro expansion (command substitution)
- Track and report kill events (projectile-type-specific carnage messages)
- Activate/deactivate input with user-provided callbacks
- Toggle Lua console mode integration

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `CommandParser` | class | Base class for registering and executing console commands. |
| `Console` | class | Singleton console extending CommandParser; adds I/O handling, history, macros, carnage reporting. |
| `command_map` | typedef | `std::map<std::string, std::function<void(const std::string&)>>` ΓÇö maps command names to handlers. |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_commands` (CommandParser) | `command_map` | instance | Maps registered command names to callback functions. |
| `m_buffer` (Console) | `std::string` | instance | Raw input buffer. |
| `m_displayBuffer` (Console) | `std::string` | instance | Formatted display version of buffer (shown to user). |
| `m_active` (Console) | `bool` | instance | Tracks whether input is currently active. |
| `m_prev_commands` (Console) | `std::vector<std::string>` | instance | Command history for up/down arrow navigation. |
| `m_macros` (Console) | `std::map<std::string, std::string>` | instance | Text macro substitutions. |
| `m_carnage_messages` (Console) | `std::vector<std::pair<std::string, std::string>>` | instance | Kill/suicide messages indexed by projectile type. |
| `environment_preferences` | `environment_preferences_data*` | global | Extern from preferences.h; checked for Lua console flag. |

## Key Functions / Methods

### CommandParser::register_command (overloaded)
- Signature: `void register_command(std::string command, std::function<void(const std::string&)> f)`; `void register_command(std::string command, const CommandParser& command_parser)`
- Purpose: Register a command with a function callback or delegate to another CommandParser.
- Inputs: Command name (string); callback function or another CommandParser.
- Outputs/Return: None.
- Side effects: Modifies `m_commands` map.
- Calls: (direct insert into map)
- Notes: Allows hierarchical command registration.

### CommandParser::parse_and_execute
- Signature: `virtual void parse_and_execute(const std::string& command_string)`
- Purpose: Parse and execute a command string by looking up the command name in `m_commands` and invoking the callback.
- Inputs: Command string.
- Outputs/Return: None.
- Side effects: Executes registered callback.
- Calls: (map lookup; invokes stored function)
- Notes: Virtual, can be overridden.

### Console::enter
- Signature: `void enter()`
- Purpose: Process the current input buffer as a command and invoke the user's callback.
- Inputs: None (uses `m_buffer`).
- Outputs/Return: None.
- Side effects: Executes callback with buffer contents; clears buffer; saves to command history.
- Calls: (inferred) `parse_and_execute` or direct callback invocation.
- Notes: Bound to Enter key.

### Console::activate_input
- Signature: `void activate_input(std::function<void (const std::string&)> callback, const std::string& prompt)`
- Purpose: Enable console input with a user callback and optional prompt text.
- Inputs: Callback function; prompt string.
- Outputs/Return: None.
- Side effects: Sets `m_active = true`; stores callback and prompt.
- Calls: None.
- Notes: Paired with `deactivate_input()`.

### Console::abort
- Signature: `void abort()`
- Purpose: Cancel input and invoke callback with empty string.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Clears buffer; calls `m_callback("")`; deactivates input.
- Calls: (inferred) callback invocation.

### Console::instance
- Signature: `static Console* instance()`
- Purpose: Return the singleton Console instance.
- Inputs: None.
- Outputs/Return: Pointer to Console.
- Side effects: None (lazy init expected in .cpp).
- Calls: None.
- Notes: Singleton accessor.

### Console keyboard event handlers
- Methods: `del()`, `backspace()`, `clear()`, `forward_clear()`, `transpose()`, `delete_word()`, `up_arrow()`, `down_arrow()`, `left_arrow()`, `right_arrow()`, `line_home()`, `line_end()`, `textEvent(const SDL_Event&)`
- Purpose: Handle individual keyboard operations (deletion, navigation, text insertion).
- Notes: Called by key-binding handlers; manipulate `m_buffer`, `m_cursor_position`, and command history.

### Console carnage reporting
- Methods: `set_carnage_message(int16 projectile_type, const std::string& on_kill, const std::string& on_suicide)`, `report_kill(int16 player_index, int16 aggressor_player_index, int16 projectile_index)`, `clear_carnage_messages()`
- Purpose: Register kill/suicide messages for projectile types and dispatch them when kills occur.
- Notes: Used for game scoreboard/chat messages.

## Control Flow Notes
The Console follows an **event-driven I/O pattern**:
1. Game engine calls keyboard event handlers (`backspace()`, `enter()`, etc.) or `textEvent()` in response to SDL input.
2. User calls `activate_input(callback, prompt)` to enable the console.
3. When user presses Enter, `enter()` executes the callback with the input string (or invokes `parse_and_execute()` internally).
4. `deactivate_input()` disables input without executing.

Command history is navigated via `up_arrow()`/`down_arrow()`, which step through `m_prev_commands`. Not inferable from header whether Lua console is integrated at initialization or runtime, but `use_lua_console()` checks `environment_preferences->use_solo_lua`.

## External Dependencies
- **Includes:** `<functional>`, `<string>`, `<map>` (STL)
- **From preferences.h:** `environment_preferences` (extern global of type `environment_preferences_data*`), checking `use_solo_lua` flag
- **SDL:** `SDL_Event` (for `textEvent()`)
- **Defined elsewhere:** `InfoTree` class; `parse_mml_console()` and `reset_mml_console()` (implementation in .cpp)
