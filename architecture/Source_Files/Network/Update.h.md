# Source_Files/Network/Update.h

## File Purpose
Defines the `Update` singleton class that manages background checking for software updates. Uses SDL threading to perform non-blocking update checks and exposes the current status and available version information to the rest of the application.

## Core Responsibilities
- Implement singleton pattern for centralized update management
- Check for new versions asynchronously in a background thread
- Track update check status (checking, failed, available, unavailable)
- Store and expose new version display string when an update is detected
- Manage SDL thread lifecycle for background operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Update` | class | Singleton update checker with threaded online version lookup |
| `Status` | enum | States: `CheckingForUpdate`, `UpdateCheckFailed`, `UpdateAvailable`, `NoUpdateAvailable` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `Update*` | static (within `instance()`) | Singleton instance holder |
| `m_status` | `Status` | member | Current update check state |
| `m_new_date_version` | `std::string` | member | New version identifier (unused in public API) |
| `m_new_display_version` | `std::string` | member | Human-readable new version string |
| `m_thread` | `SDL_Thread*` | member | Active background thread pointer |

## Key Functions / Methods

### `instance()`
- Signature: `static Update* instance()`
- Purpose: Lazy-initialize and return singleton instance
- Inputs: None
- Outputs/Return: Pointer to static `Update` instance
- Side effects: Creates instance on first call
- Calls: Constructor (implicit)
- Notes: Thread-unsafe; assumes single initialization context

### `GetStatus()`
- Signature: `Status GetStatus()`
- Purpose: Query current update check status
- Inputs: None
- Outputs/Return: `Status` enum value
- Side effects: None
- Calls: None
- Notes: Non-blocking; returns immediately

### `NewDisplayVersion()`
- Signature: `std::string NewDisplayVersion()`
- Purpose: Retrieve human-readable new version string
- Inputs: None
- Outputs/Return: Display version string
- Side effects: None
- Calls: None
- Notes: Asserts `m_status == UpdateAvailable`; caller must verify status first

### `StartUpdateCheck()`
- Signature: `void StartUpdateCheck()`
- Purpose: Initiate background update check thread
- Inputs: None
- Outputs/Return: None
- Side effects: Spawns SDL thread, updates `m_status`, assigns `m_thread`
- Calls: `SDL_CreateThread` (likely), `update_thread`
- Notes: Not inferable from header; implementation in .cpp

### `update_thread()` (static)
- Signature: `static int update_thread(void*)`
- Purpose: SDL thread entry point for update checking
- Inputs: `void*` opaque pointer (likely `this`)
- Outputs/Return: SDL thread exit code
- Side effects: Calls `Thread()` on instance
- Calls: `Thread`
- Notes: Standard SDL thread signature; wrapper around instance method

### `Thread()`
- Signature: `int Thread()`
- Purpose: Instance-level update check logic (runs in background thread)
- Inputs: None (state via member variables)
- Outputs/Return: Exit code
- Side effects: Updates `m_status` and `m_new_display_version` asynchronously
- Calls: Not inferable from header
- Notes: Implementation details unknown; likely performs HTTP request to check remote version

## Control Flow Notes
This subsystem is optional and likely started during game initialization or on-demand. It operates entirely out-of-band via SDL threading:
1. Main thread calls `StartUpdateCheck()` or waits for external trigger
2. Worker thread performs blocking I/O (HTTP fetch) without stalling the game loop
3. Worker updates `m_status` and version strings
4. Main thread polls `GetStatus()` / `NewDisplayVersion()` to decide UI feedback
5. No explicit frame-loop integration; status changes are consumed by polling

## External Dependencies
- `<string>` ΓÇö Standard library strings for version storage
- `<SDL2/SDL_thread.h>` ΓÇö Cross-platform threading primitives
- `cseries.h` ΓÇö Aleph One utilities (likely macros, type defs, `assert`)
- Network code (not visible; likely internal HTTP library) ΓÇö called by `Thread()` implementation
