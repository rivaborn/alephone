# Source_Files/CSeries/csalerts_sdl.cpp

## File Purpose

Implements SDL-based alert dialogs, debugging support, and error handling for the Aleph One game engine. Provides platform-specific (Windows, macOS, Linux) implementations for displaying user alerts, launching URLs, selecting scenarios, and handling assertions/fatal errors with logging integration.

## Core Responsibilities

- Display alert messages in-game (via SDL dialogs) or via system dialogs when game screen unavailable
- Handle assertion failures and runtime warnings with file/line context
- Manage fatal errors with graceful shutdown and logging
- Provide scenario selection UI via native folder browser (Windows)
- Launch URLs in default browser (cross-platform)
- Integrate debug events with centralized logging system

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `dialog` | class (from sdl_dialogs.h) | Modal dialog container for in-game alerts |
| `vertical_placer` | class (from sdl_dialogs.h) | Vertical layout manager for dialog widgets |
| `w_static_text` | class (from sdl_widgets.h) | Text label widget for message display |
| `w_button` | class (from sdl_widgets.h) | Clickable button widget |
| `BROWSEINFOW` | struct (Windows API) | Folder browser configuration |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `assert_text` | `char[256]` | file-static | Buffer for formatting assert/warning messages with file:line context |

## Key Functions / Methods

### system_alert_user
- **Signature:** `void system_alert_user(const char* message, short severity)` (non-macOS only)
- **Purpose:** Display native system alert dialog (Windows) or stderr output (Linux)
- **Inputs:** message string, severity level (infoError or fatalError)
- **Outputs/Return:** None (void)
- **Side effects:** Shows system dialog or writes to stderr; may block on dialog interaction
- **Calls:** `utf8_to_wide()` (Windows), `fprintf()` (Linux), `MessageBoxW()` (Windows)
- **Notes:** Platform-conditional; macOS implemented separately (extern declaration)

### system_alert_choose_scenario
- **Signature:** `bool system_alert_choose_scenario(char *chosen_dir)` (Windows only)
- **Purpose:** Open native folder browser for scenario selection; return chosen directory path
- **Inputs:** pointer to 256-byte char buffer for result
- **Outputs/Return:** true if user selected folder, false on cancel/error; writes path to buffer
- **Side effects:** Allocates/deallocates COM objects (LPITEMIDLIST, LPMALLOC)
- **Calls:** `SHBrowseForFolderW()`, `SHGetPathFromIDListW()`, `SHGetMalloc()`, `WideCharToMultiByte()`
- **Notes:** Windows-specific; initializes browser with current working directory via callback

### browse_callback_proc
- **Signature:** `static int CALLBACK browse_callback_proc(HWND hwnd, UINT msg, LPARAM lparam, LPARAM lpdata)` (Windows only)
- **Purpose:** Win32 callback to set initial folder in SHBrowseForFolder dialog
- **Inputs:** window handle, message code, two LPARAM parameters
- **Outputs/Return:** 0 (required by Win32 callback protocol)
- **Side effects:** None (reads current directory, sends window messages)
- **Calls:** `GetCurrentDirectoryW()`, `SendMessageW()`
- **Notes:** Responds only to BFFM_INITIALIZED message; silently ignores others

### system_launch_url_in_browser
- **Signature:** `void system_launch_url_in_browser(const char *url)` (non-macOS only)
- **Purpose:** Open URL in default browser (Windows: ShellExecute, Linux: fork/execlp)
- **Inputs:** URL string
- **Outputs/Return:** None (void)
- **Side effects:** Spawns external process (browser); on Linux, fork() + exec family calls
- **Calls:** `ShellExecuteW()` (Windows), `fork()`, `execlp()` (Linux)
- **Notes:** Linux tries xdg-open first, fallback to sensible-browser; parent waits via `wait()`

### alert_user (string variant)
- **Signature:** `void alert_user(const char *message, short severity)` (primary variant)
- **Purpose:** Display in-game or system alert dialog with text wrapping; honors game state
- **Inputs:** message string, severity level (fatalError, infoNoError, infoError)
- **Outputs/Return:** None (void); exits process if severity==fatalError
- **Side effects:** Shows modal dialog if game visible; calls `update_game_window()` after dialog; exits on fatal error; logs message
- **Calls:** `MainScreenVisible()`, `SDL_ShowSimpleMessageBox()`, `get_theme_font()`, `text_width()`, `strdup()`, `free()`, `update_game_window()`, logging macros
- **Notes:** Wraps long text to MAX_ALERT_WIDTH; allocates dynamic dialog widgets; `#ifndef A1_NETWORK_STANDALONE_HUB` conditional

### alert_user (resource variant)
- **Signature:** `void alert_user(short severity, short resid, short item, int error)`
- **Purpose:** Display alert from string resource with error code appended
- **Inputs:** severity, resource ID, item ID, error code
- **Outputs/Return:** None (void)
- **Side effects:** Logs error via `logError()` or `logFatal()` before showing alert
- **Calls:** `getcstr()`, `sprintf()`, logging macros, `alert_user(const char*, short)`
- **Notes:** Formats message as "resource_string (error %d)"

### vhalt
- **Signature:** `void vhalt(const char *message)` 
- **Purpose:** Log fatal error message, flush logs, shutdown gracefully, display alert, and abort
- **Inputs:** error message
- **Outputs/Return:** Does not return (calls `abort()`)
- **Side effects:** Calls `stop_recording()`, logs fatal error, flushes logger, calls `shutdown_application()`, shows system alert, calls `abort()`
- **Calls:** `stop_recording()`, `logFatal()`, `GetCurrentLogger()->flush()`, `shutdown_application()`, `system_alert_user()`, `abort()`
- **Notes:** Called on unrecoverable errors; ensures recording stopped and logs flushed before exit

### _alephone_assert
- **Signature:** `void _alephone_assert(const char *file, int32 line, const char *what)`
- **Purpose:** Handle assertion failure with file:line context; always fatal
- **Inputs:** source filename, line number, assertion expression
- **Outputs/Return:** Does not return
- **Side effects:** Formats message to `assert_text`, calls `vhalt()` ΓåÆ abort
- **Calls:** `csprintf()`, `vhalt()`
- **Notes:** Intended for compile-time macro `assert()`; message format: "file:line: condition"

### _alephone_warn
- **Signature:** `void _alephone_warn(const char *file, int32 line, const char *what)`
- **Purpose:** Handle runtime warning with file:line context; non-fatal
- **Inputs:** source filename, line number, warning message
- **Outputs/Return:** None (void)
- **Side effects:** Formats message to `assert_text`, calls `vpause()` ΓåÆ logs + stderr
- **Calls:** `csprintf()`, `vpause()`
- **Notes:** Parallel to _alephone_assert but does not exit

## Control Flow Notes

**Error Hierarchy:**
1. **Fatal errors** ΓåÆ `vhalt()` ΓåÆ `shutdown_application()` + system alert + `abort()`
2. **Non-fatal errors** ΓåÆ `alert_user()` ΓåÆ in-game or system dialog (depending on `MainScreenVisible()`) ΓåÆ continue
3. **Warnings** ΓåÆ `vpause()` + stderr ΓåÆ continue (debug only)

**Alert Display Logic:**
- If game screen **not visible**: Use `SDL_ShowSimpleMessageBox()` (system dialog)
- If game screen **visible**: Build in-game dialog with `vertical_placer`, text wrapping, button; call `d.run()` (modal)
- Post-dialog: call `update_game_window()` if not fatal

**Shutdown Path:**
- Assertions/vhalt calls trigger `stop_recording()` (record cleanup) ΓåÆ `logFatal()` (log) ΓåÆ `flush()` (ensure log written) ΓåÆ `shutdown_application()` (engine cleanup) ΓåÆ system alert ΓåÆ `abort()`

## External Dependencies

- **SDL2** (`SDL.h`): `SDL_ShowSimpleMessageBox()`, `SDL_MESSAGEBOX_*` constants
- **Windows API**: `windows.h`, `tchar.h`, `shellapi.h`, `shlobj.h` for folder browser and shell execution
- **POSIX/Unix**: `sys/wait.h` for `wait()`, `fork()`, `execlp()`
- **Game Engine** (defined elsewhere):
  - `cseries.h`: Core includes (types, macros, string utilities)
  - `Logging.h`: `GetCurrentLogger()`, `logFatal()`, `logError()`, `logWarning()`, `logNote()`
  - `sdl_dialogs.h`: `dialog`, `vertical_placer`, `top_dialog`
  - `sdl_widgets.h`: `w_title`, `w_static_text`, `w_button`, `w_spacer`
  - Platform-specific: `update_game_window()`, `MainScreenVisible()`, `stop_recording()`, `shutdown_application()` (defined elsewhere)
  - Resource/string management: `getcstr()`, `csprintf()`, `text_width()`, `get_theme_font()`

---

**Notes on Terminology:**
- "functions" = C++ free functions (no methods in classes defined here)
- Distinction between in-game dialogs (custom UI system) vs. system dialogs (native OS)
- Heavy use of conditional compilation (`#ifdef __MACOSX__`, `#if defined(__WIN32__)`) for platform abstraction
