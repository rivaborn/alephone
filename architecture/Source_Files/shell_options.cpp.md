# Source_Files/shell_options.cpp

## File Purpose
Implements command-line argument parsing for Aleph One (a game engine). Processes flags, commands, and options from argc/argv, populating a global `ShellOptions` struct with parsed values. Handles help/version display and validates arguments.

## Core Responsibilities
- Parse command-line arguments and categorize them (commands, flags, string options, files/directories)
- Match arguments against registered options using short/long names
- Execute command callbacks (ΓÇôhelp, ΓÇôversion) which may exit the process
- Validate and report unrecognized arguments
- Generate formatted help/usage output with aligned columns
- Handle platform-specific output (Windows MessageBox vs. stdout)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ShellOptionsOption | struct | Base for all option types; matches short/long names against argument strings |
| ShellOptionsCommand | struct | Option that triggers a callback function (e.g., help, version) |
| ShellOptionsFlag | struct | Option that sets a bool reference when matched |
| ShellOptionsString | struct | Option that consumes the next argument as a string value |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| shell_options | ShellOptions | global extern | Global instance populated by parse(); holds all parsed options |
| shell_options_commands | std::vector | static | Registered commands: help (ΓÇôh), version (ΓÇôv) |
| shell_options_flags | std::vector | static | Registered boolean flags: debug, fullscreen, windowed, nogl, nosound, nogamma, nojoystick, insecure_lua, skip_intro, editor, no_chooser |
| shell_options_strings | std::vector | static | Registered string options: output (ΓÇôo), replay_directory (ΓÇôl), NSDocumentRevisionsDebugMode (Xcode debug arg) |
| help_tab_stop | int | static | Column position (33) for right-aligning help text |
| ignore | std::string | static | Dummy string reference for discarding unwanted arguments (e.g., Xcode flags) |

## Key Functions / Methods

### ShellOptions::parse
- **Signature:** `std::unordered_map<int, bool> parse(int argc, char** argv, bool ignore_unknown_args = false)`
- **Purpose:** Main entry point; parses all command-line arguments and populates the global `shell_options` struct.
- **Inputs:** argc, argv (standard C main parameters); ignore_unknown_args flag
- **Outputs/Return:** Map of argument indices ΓåÆ whether they were recognized/consumed
- **Side effects:** Modifies global `shell_options` members; calls `exit(0)` for commands; calls `exit(1)` on fatal error; logs to Logging and stdout/stderr
- **Calls:** FileSpecifier::Exists(), FileSpecifier::IsDir(), logFatal(), printf(), exit(), command callbacks (print_usage/print_version)
- **Notes:** 
  - Handles `-C` prefix by stripping it (likely for internal use)
  - Commands exit immediately; other options accumulate
  - Files/directories are detected by filesystem checks, not syntax
  - Unknown arguments cause fatal error unless `ignore_unknown_args=true`

### print_usage
- **Signature:** `static void print_usage()`
- **Purpose:** Display formatted help message with all commands, flags, and options.
- **Inputs:** None (reads global `shell_options_commands`, `shell_options_flags`, `shell_options_strings`)
- **Outputs/Return:** Prints to stdout (Unix) or MessageBox (Windows)
- **Side effects:** Formatted output; platform-specific display
- **Calls:** operator<< overload, spaces(), utf8_to_wide() (Windows)
- **Notes:** Help text is aligned at column 33; directories and files are documented at the end

### print_version
- **Signature:** `static void print_version()`
- **Purpose:** Display application name and version.
- **Inputs:** None
- **Outputs/Return:** Prints to stdout
- **Side effects:** I/O
- **Calls:** expand_app_variables(), std::cout
- **Notes:** Expands variable `$appLongVersion$` in the string "Aleph One $appLongVersion$"

## Control Flow Notes
**Initialization/Startup:** Called early in main() to populate `shell_options` before engine initialization.

**Parse loop:** For each argument:
1. Commands are checked first and immediately exit(0) if matched
2. Flags are checked and set to true without consuming additional arguments
3. String options are checked and consume the next argument
4. Non-option arguments (no leading `ΓÇô`) are tested as files/directories
5. Unmatched arguments are logged as fatal and exit(1) unless `ignore_unknown_args=true`

**Output:** Help/version messages are generated on-demand and may exit the process.

## External Dependencies
- **Standard library:** `<iostream>`, `<functional>`, `<sstream>`, `<string>`, `<vector>`, `<unordered_map>`
- **Platform:** `<windows.h>` (Windows only, for MessageBoxW)
- **Project headers:** 
  - `shell_options.h` (struct definition, extern global)
  - `FileHandler.h` (FileSpecifier for file/directory checks)
  - `Logging.h` (logFatal macro)
  - `csstrings.h` (expand_app_variables, utf8_to_wide for Windows)
