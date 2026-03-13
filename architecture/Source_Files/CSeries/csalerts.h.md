# Source_Files/CSeries/csalerts.h

## File Purpose
Header providing alert/error handling utilities for the Aleph One game engine. Declares functions for displaying user-facing error dialogs, fatal error termination, and debug assertion/warning macros. Serves as the engine's centralized error reporting and debugging interface.

## Core Responsibilities
- Display informational and error alerts to the user with optional resource string formatting
- Handle fatal errors with program termination (via `halt()`)
- Provide debug assertion and warning macros that are compiled out in release builds
- Generate context-specific fatal alerts (out of memory, corrupted files, bad data)
- Support scenario selection dialogs and URL launching
- Control debug flow with `pause_debug()` and `vpause()` for breakpoint-style debugging

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (enum, unnamed) | enum | Severity levels: `infoError`, `fatalError`, `infoNoError` |

## Global / File-Static State
None.

## Key Functions / Methods

### alert_user (overload 1)
- Signature: `void alert_user(const char *message, short severity = infoError)`
- Purpose: Display a simple string message to the user with specified severity level
- Inputs: C-string message, severity (defaults to `infoError`)
- Outputs/Return: void
- Side effects: Displays UI dialog to user
- Calls: (external; implementation not in this file)
- Notes: Default parameter uses C++ syntax in a C header, suggesting C++ compilation

### alert_user (overload 2)
- Signature: `void alert_user(short severity, short resid, short item, int error)`
- Purpose: Display a formatted alert using resource strings identified by `resid` and `item`, optionally appending an error code
- Inputs: severity level, resource ID, item ID, error code
- Outputs/Return: void
- Side effects: Displays UI dialog to user
- Calls: (external; implementation not in this file)
- Notes: Resource-driven approach for localized/templated messages

### alert_user_fatal
- Signature: `static inline void alert_user_fatal(short resid, short item, int error) NORETURN`
- Purpose: Display a fatal error alert and terminate the program
- Inputs: resource ID, item ID, error code
- Outputs/Return: Never returns (marked `NORETURN`)
- Side effects: Calls `alert_user()` then `halt()` (program termination)
- Calls: `alert_user()`, `halt()`
- Notes: Inline convenience wrapper; marked with `NORETURN` attribute for compiler optimization

### alert_out_of_memory
- Signature: `static inline void alert_out_of_memory(int error = -1) NORETURN`
- Purpose: Display out-of-memory fatal error and exit
- Inputs: error code (defaults to -1)
- Outputs/Return: Never returns
- Side effects: Program termination
- Calls: `alert_user_fatal(128, 14, error)`
- Notes: Hardcoded resource IDs (128=resource section, 14=item)

### alert_bad_extra_file
- Signature: `static inline void alert_bad_extra_file(int error = -1) NORETURN`
- Purpose: Display corrupt/missing extra file fatal error and exit
- Inputs: error code (defaults to -1)
- Outputs/Return: Never returns
- Side effects: Program termination
- Calls: `alert_user_fatal(128, 5, error)`

### alert_corrupted_map
- Signature: `static inline void alert_corrupted_map(int error = -1) NORETURN`
- Purpose: Display corrupted map file fatal error and exit
- Inputs: error code (defaults to -1)
- Outputs/Return: Never returns
- Side effects: Program termination
- Calls: `alert_user_fatal(128, 23, error)`

### halt
- Signature: `void halt(void) NORETURN`
- Purpose: Terminate program immediately
- Inputs: none
- Outputs/Return: Never returns
- Side effects: Program termination
- Calls: (external; implementation not in this file)

### vhalt
- Signature: `void vhalt(const char *message) NORETURN`
- Purpose: Terminate program with a debug message
- Inputs: message string
- Outputs/Return: Never returns
- Side effects: Program termination (optionally logs message)
- Calls: (external; implementation not in this file)

### _alephone_assert
- Signature: `void _alephone_assert(const char *file, int32 line, const char *what) NORETURN`
- Purpose: Handle assertion failure; implementation calls termination
- Inputs: source file path, line number, assertion description string
- Outputs/Return: Never returns (terminates after logging)
- Side effects: Logs/displays assertion failure, terminates program
- Calls: (external; implementation not in this file)

### _alephone_warn
- Signature: `void _alephone_warn(const char *file, int32 line, const char *what)`
- Purpose: Log/display a warning without terminating
- Inputs: source file path, line number, warning description
- Outputs/Return: void
- Side effects: Logs/displays warning, program continues
- Calls: (external; implementation not in this file)

**Notes on helper functions:**
- `pause_debug()` / `vpause(const char *message)`: Pause execution for debugger (implementation external)
- `alert_choose_scenario(char *chosen_dir)`: File picker dialog for scenario selection (implementation external)
- `launch_url_in_browser(const char *url)`: Open URL in system browser (implementation external)

## Control Flow Notes
This header is not part of the main game loop. Functions are called reactively when errors occur, resource loading fails, assertions trigger, or user interaction is needed. In DEBUG builds, assertion macros (`assert`, `vassert`, `warn`, `vwarn`) are active; in release builds they compile to no-ops. The `fc_assert` family is disabled by default, enabled only if `DEBUG_FAST_CODE` is defined, suggesting use in performance-critical code paths.

## External Dependencies
- Includes: `cstypes.h` (for `int32` typedef)
- Compiler-specific: `__attribute__((noreturn))` for GCC; fallback no-op for other compilers via `NORETURN` macro
- External symbols: all function implementations and the resource system (resid/item pairs) are defined elsewhere
