# Source_Files/CSeries/cspaths_sdl.cpp

## File Purpose
Implements platform-specific file path resolution for the Aleph One game engine across Windows, macOS, and Linux. Provides standardized access to user directories (documents, preferences, cached data, save games) and application metadata, abstracting away OS-specific APIs and conventions.

## Core Responsibilities
- Return OS-appropriate paths for game data, preferences, logs, screenshots, and save files
- Resolve user home/documents directories using platform APIs (Windows shell folders, Linux environment variables)
- Cache resolved paths to avoid repeated OS calls
- Provide platform-specific path list separators (`;` on Windows, `:` on Unix)
- Supply application identification strings (display name, reverse-domain identifier)
- Fall back safely when OS queries fail (e.g., invalid usernames, missing HOME env var)

## Key Types / Data Structures
None.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `local_dir` | `std::string` | function-static (in `_get_local_data_path`) | Cached user data directory path |
| `default_dir` | `std::string` | function-static (in `_get_default_data_path`) | Cached executable parent directory |
| `prefs_dir` | `std::string` | function-static (in `_get_prefs_path`) | Cached preferences directory path |
| `login_name` | `std::string` | function-static (in `_get_legacy_login_name`) | Cached legacy Windows username |

## Key Functions / Methods

### get_path_list_separator
- **Signature:** `char get_path_list_separator()`
- **Purpose:** Return platform-specific path list delimiter (`;` on Windows; `:` on macOS/Linux).
- **Inputs:** None.
- **Outputs/Return:** Character: `;` or `:`.
- **Side effects:** None.
- **Calls:** None.

### _get_local_data_path (Windows)
- **Signature:** `static std::string _get_local_data_path()`
- **Purpose:** Resolve user's local documents directory and append `\AlephOne`.
- **Inputs:** None.
- **Outputs/Return:** Cached path string (e.g., `C:\Users\Username\Documents\AlephOne`).
- **Side effects:** Caches result in static `local_dir`; queries `SHGetFolderPathW` on first call.
- **Calls:** `SHGetFolderPathW`, `wide_to_utf8`.
- **Notes:** Creates directory if missing (`CSIDL_FLAG_CREATE`); subsequent calls return cached value.

### _get_default_data_path (Windows)
- **Signature:** `static std::string _get_default_data_path()`
- **Purpose:** Resolve parent directory of the running executable.
- **Inputs:** None.
- **Outputs/Return:** Cached executable parent directory path; empty string on failure.
- **Side effects:** Caches result in static `default_dir`; queries `GetModuleFileNameW` on first call.
- **Calls:** `GetModuleFileNameW`, `wcsrchr`, `wide_to_utf8`.
- **Notes:** Returns empty if exe path cannot be determined. Truncation risk if path exceeds `MAX_PATH`.

### _get_prefs_path (Windows)
- **Signature:** `static std::string _get_prefs_path()`
- **Purpose:** Resolve user's local AppData directory and append `\AlephOne`.
- **Inputs:** None.
- **Outputs/Return:** Cached preferences path (e.g., `C:\Users\Username\AppData\Local\AlephOne`).
- **Side effects:** Caches result in static `prefs_dir`; queries `SHGetFolderPathW` on first call.
- **Calls:** `SHGetFolderPathW`, `wide_to_utf8`.
- **Notes:** Creates directory if missing; typical location for user settings.

### _get_legacy_login_name (Windows)
- **Signature:** `static std::string _get_legacy_login_name()`
- **Purpose:** Retrieve Windows username for constructing legacy save paths.
- **Inputs:** None.
- **Outputs/Return:** Cached username string (max ~16 chars); fallback `"Bob User"` if invalid.
- **Side effects:** Caches result in static `login_name`; queries `GetUserNameA` on first call.
- **Calls:** `GetUserNameA`, `strcpy`, `strpbrk`.
- **Notes:** Rejects usernames containing path-invalid characters (`\ / : * ? " < > |`). Fixed 17-char buffer.

### get_data_path (Windows)
- **Signature:** `std::string get_data_path(CSPathType type)`
- **Purpose:** Main entry point; return platform-appropriate path for a given path type.
- **Inputs:** `CSPathType` enum (kPathLocalData, kPathPreferences, kPathSavedGames, etc.).
- **Outputs/Return:** Resolved absolute path string; empty if not applicable.
- **Side effects:** None directly (helpers manage caching).
- **Calls:** `_get_local_data_path`, `_get_default_data_path`, `_get_prefs_path`, `_get_legacy_login_name`.
- **Notes:** kPathBundleData and kPathLegacyData not supported (return empty). Uses `\` separators. Constructs compound paths by concatenating base + subdirectories (e.g., `Screenshots`, `Saved Games`).

### get_data_path (Linux)
- **Signature:** `std::string get_data_path(CSPathType type)`
- **Purpose:** Main entry point; return platform-appropriate path for a given path type.
- **Inputs:** `CSPathType` enum.
- **Outputs/Return:** Resolved path string; empty if not applicable or HOME unset.
- **Side effects:** None directly.
- **Calls:** `_get_local_data_path`.
- **Notes:** kPathDefaultData uses compile-time `PKGDATADIR` macro. Most paths rooted at `~/.alephone`. Uses `/` separators. Legacy and bundle paths not applicable.

### get_application_name / get_application_identifier
- **Signature:** `std::string get_application_name()`, `std::string get_application_identifier()`
- **Purpose:** Return display name (`"Aleph One"`) and reverse-domain identifier (`"org.bungie.source.AlephOne"`).
- **Inputs:** None.
- **Outputs/Return:** Hardcoded strings (from macros and literals).
- **Side effects:** None.
- **Calls:** None.

## Control Flow Notes
File is platform-specific: only one of the three implementations (Apple/macOS, Windows, or Linux) is compiled based on preprocessor conditionals. Initialization typically calls `get_data_path(kPathLocalData)` and similar during engine startup to set up directory structures. Static caching ensures repeated path queries are O(1) after the first call.

## External Dependencies
- **Windows-specific:** `<windows.h>`, `<shlobj.h>` (shell folder APIs); `SHGetFolderPathW`, `GetModuleFileNameW`, `GetUserNameA`.
- **Cross-platform standard:** `<string>`, `getenv()` (POSIX), `strcpy`, `strpbrk`.
- **Internal:** `cstypes.h` (type aliases), `cspaths.h` (enum CSPathType, function declarations), `csstrings.h` (wide_to_utf8 conversion), `alephversion.h` (A1_DISPLAY_NAME macro).
- **Optional:** `confpaths.h` (PKGDATADIR compile-time constant for Linux package installations).
