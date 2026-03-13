# Source_Files/CSeries/cspaths.h

## File Purpose
Provides a cross-platform interface for accessing OS-specific directory paths (data, preferences, logs, etc.) and application metadata. Part of the Aleph One game engine's core utilities for abstracting filesystem operations.

## Core Responsibilities
- Define enumerated path types for application directories (data, preferences, logs, screenshots, saved games, etc.)
- Provide function to retrieve platform-specific filesystem paths by type
- Supply application name and identifier getters
- Abstract platform-specific path separators and conventions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CSPathType | enum | Enumeration of directory types: local data, defaults, legacy locations, preferences, logs, screenshots, saved games, cache, recordings |

## Global / File-Static State
None.

## Key Functions / Methods

### get_data_path
- Signature: `std::string get_data_path(CSPathType type);`
- Purpose: Retrieve the filesystem path for a given directory type
- Inputs: `CSPathType type` ΓÇô enum specifying which directory path to return
- Outputs/Return: `std::string` containing the absolute or relative path
- Side effects: Implementation likely performs platform-specific system calls; may create directories
- Calls: Not visible from this file (declaration only)
- Notes: Behavior varies by OS and path type; handles legacy path migration

### get_path_list_separator
- Signature: `char get_path_list_separator();`
- Purpose: Return the platform-appropriate character for separating multiple paths in a list
- Inputs: None
- Outputs/Return: `char` (`:` on Unix-like systems, `;` on Windows)
- Side effects: None
- Calls: Not visible from this file
- Notes: Used when constructing or parsing colon/semicolon-delimited PATH-like strings

### get_application_name
- Signature: `std::string get_application_name();`
- Purpose: Return the human-readable application name
- Inputs: None
- Outputs/Return: `std::string` (e.g., "Aleph One")
- Side effects: None
- Calls: Not visible from this file
- Notes: May be used for window titles, UI dialogs, user-facing messages

### get_application_identifier
- Signature: `std::string get_application_identifier();`
- Purpose: Return the application's unique identifier for the platform
- Inputs: None
- Outputs/Return: `std::string` (e.g., reverse domain notation on macOS)
- Side effects: None
- Calls: Not visible from this file
- Notes: Used for preferences storage, file associations, system registration

## Control Flow Notes
This is a pure utility header providing initialization-time and runtime path/metadata queries. Not part of the frame loop; called during engine startup to configure directory structures and during file I/O operations throughout execution.

## External Dependencies
- `cstypes.h` ΓÇô base integer types and constants
- `<string>` ΓÇô C++ standard string library
- Implementation file requires platform-specific includes (e.g., `<windows.h>`, `<unistd.h>`, `<pwd.h>`)
