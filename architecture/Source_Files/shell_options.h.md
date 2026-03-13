# Source_Files/shell_options.h

## File Purpose
Defines command-line option parsing and storage for the game shell/launcher. The `ShellOptions` struct aggregates all runtime flags and configuration that can be passed via command-line arguments, including rendering, audio, input, and gameplay settings.

## Core Responsibilities
- Define the `ShellOptions` struct to store parsed command-line options
- Provide a `parse()` method to populate the struct from argc/argv
- Store boolean feature flags (graphics, sound, input, debug modes)
- Store file paths (replay directory, working directory, output, input files)
- Support optional unknown argument handling during parsing
- Expose a global `shell_options` instance for engine-wide access

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ShellOptions` | struct | Container for all shell/launcher configuration flags and file paths |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `shell_options` | `ShellOptions` | global (extern) | Singleton instance accessible engine-wide after initialization |

## Key Functions / Methods

### parse
- **Signature:** `std::unordered_map<int, bool> parse(int argc, char** argv, bool ignore_unknown_args = false)`
- **Purpose:** Parse command-line arguments into `ShellOptions` fields
- **Inputs:** `argc`/`argv` (standard C program arguments); `ignore_unknown_args` (whether to suppress errors for unrecognized flags)
- **Outputs/Return:** `std::unordered_map<int, bool>` (likely mapping argument indices to parsing success/error status)
- **Side effects:** Modifies member variables (`program_name`, boolean flags, `directory`, `files`, etc.)
- **Calls:** Not inferable from header
- **Notes:** Default parameter allows lenient parsing; implementation likely in corresponding `.cpp` file

## Control Flow Notes
This file supports **initialization phase**: command-line parsing occurs early in engine startup before main systems initialize, based on the presence of flags like `skip_intro`, `editor`, and `debug` that gate subsequent initialization.

## External Dependencies
- **Standard Library:** `<string>`, `<vector>`, `<unordered_map>` (STL containers)
- **Defined elsewhere:** `parse()` implementation (likely `shell_options.cpp`)
