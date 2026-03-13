# Source_Files/CSeries/csalerts_sdl.cpp - Enhanced Analysis

## Architectural Role

This file bridges cross-platform error handling, fatal shutdown coordination, and user notification in the Aleph One engine. It implements a **dual-modal alert system** (system dialogs vs. in-game dialogs) that respects game state, while also serving as a **critical shutdown coordinator** that sequences `stop_recording()` ΓåÆ `logFatal()` ΓåÆ `flush()` ΓåÆ `shutdown_application()` ΓåÆ `abort()`. The heavy platform-specific compilation (`#ifdef __WIN32__`, `#ifdef __MACOSX__`, else Linux) makes it the **abstraction boundary for native OS dialogs**; macOS code is stubbed via extern, while Windows/Linux are inlined. This design isolates the engine core from OS particulars while ensuring graceful, logged shutdown on unrecoverable errors.

## Key Cross-References

### Incoming (who depends on this)
- **Assertion/warning macros** in `cseries.h` call `_alephone_assert()` and `_alephone_warn()` (cross-ref: `Source_Files/CSeries/csalerts.h`)
- **Game loop (`marathon2.cpp`)** calls `alert_user()` for runtime errors (map corruption, OOM)
- **Network/rendering subsystems** call `vhalt()` on fatal conditions (state invariant violations)
- **Test/startup code** calls `alert_choose_scenario()` for scenario selection

### Outgoing (what this file depends on)
- **Logging subsystem** (`Logging.h`): calls `logFatal()`, `logError()`, `logWarning()`, `logNote()`, `GetCurrentLogger()->flush()`
- **Game state queries** (`MainScreenVisible()`, `update_game_window()`) ΓÇö state from rendering/shell subsystems
- **Shutdown coordination** (`stop_recording()` from Videos subsystem, `shutdown_application()` from Shell)
- **Widget framework** (`dialog`, `vertical_placer`, `w_title`, `w_static_text`, `w_button`, `w_spacer` from `sdl_dialogs.h` / `sdl_widgets.h`)
- **Font/theme system** (`get_theme_font()`, `text_width()`) for text layout
- **Resource strings** (`getcstr()`) for localized alert messages
- **Windows API** for native dialogs (`SHBrowseForFolderW`, `MessageBoxW`, `ShellExecuteW`)
- **POSIX** for subprocess spawning (`fork()`, `execlp()`, `wait()`)

## Design Patterns & Rationale

### 1. Platform Abstraction via Extern + Conditional Compilation
macOS implementations are **declared extern** (stubbed in platform layer), while Windows/Linux inlining conditional code within the same function. This defers macOS-specific dialog code to `Cocoa` layer while allowing Windows (native API) and Linux (shell scripts) to coexist.

```cpp
#ifdef __MACOSX__
extern void system_alert_user(const char*, short);  // Implemented in platform layer
#else
void system_alert_user(...) { /* Win32 or Linux */ }
```

Why: Maintains single source while letting platform teams own UI/OS-specific code paths.

### 2. Dual-Modal Error Presentation
`alert_user()` checks `MainScreenVisible()` and branches:
- **Screen not visible** ΓåÆ Native OS dialog (blocking, simple)
- **Screen visible** ΓåÆ In-game widget dialog (respects game rendering context, themeable)

Why: Errors during startup/shutdown use system dialogs; in-game errors stay on-theme and don't interrupt rendering cycle.

### 3. Graceful Shutdown Sequence
`vhalt()` implements **ordered cleanup** before exit:
```
stop_recording() ΓåÆ logFatal() ΓåÆ flush() ΓåÆ shutdown_application() ΓåÆ alert ΓåÆ abort()
```
Why: Recording must stop before logs are flushed (to avoid mid-write corruption); app shutdown cleans game state; alert is shown after cleanup to avoid re-entrancy.

### 4. Manual Text Wrapping with Font Metrics
`alert_user()` manually wraps text by finding the last space within `MAX_ALERT_WIDTH` pixel boundary using `text_width()`:
```cpp
while (i < strlen(t) && width < MAX_ALERT_WIDTH) {
  width = text_width(t, i, font, style);
  if (t[i] == ' ') last = i;
  i++;
}
```
Why: No modern text layout library; integrates with engine's font system and theming. Respects word boundaries.

## Data Flow Through This File

```
ΓöîΓöÇ User Error / Assertion ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÉ
Γöé                                                Γöé
v                                                Γöé
_alephone_assert() / _alephone_warn()            Γöé
  ΓööΓöÇ csprintf(assert_text, "file:line: msg")     Γöé
    ΓööΓöÇ vhalt() [fatal] or vpause() [warn]        Γöé
      Γöé                                           Γöé
      Γö£ΓöÇ> logFatal() ΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöÇΓöñ
      Γöé   logWarning() / logNote()                Γöé
      Γöé                                           Γöé
      Γö£ΓöÇ> GetCurrentLogger()->flush()             Γöé
      Γöé   (ensure logs written to disk)           Γöé
      Γöé                                           Γöé
      Γö£ΓöÇ> shutdown_application() [fatal only]     Γöé
      Γöé   (clean game state, close files)         Γöé
      Γöé                                           Γöé
      Γö£ΓöÇ> system_alert_user() [fatal only]        Γöé
      Γöé   or alert_user() [non-fatal]             Γöé
      Γöé                                           Γöé
      Γö£ΓöÇ> MainScreenVisible()                     Γöé
      Γöé   Γö£ΓöÇ Yes: show in-game dialog             Γöé
      Γöé   Γöé   ΓööΓöÇ wrap text ΓåÆ display w/ font      Γöé
      Γöé   Γöé     ΓööΓöÇ user clicks OK                 Γöé
      Γöé   Γöé       ΓööΓöÇ update_game_window()         Γöé
      Γöé   Γöé                                        Γöé
      Γöé   ΓööΓöÇ No: SDL_ShowSimpleMessageBox()       Γöé
      Γöé       (native OS modal)                   Γöé
      Γöé                                           Γöé
      ΓööΓöÇ> abort() [fatal] or return [non-fatal]   Γöé
```

## Learning Notes

1. **Era-appropriate text layout:** Manual word-wrapping with binary search for space position is typical of early 2000s game engines before rich text libraries became standard. Modern engines use FreeType, HarfBuzz, or integrated layout systems.

2. **Logging was post-hoc:** Comments indicate logging integration was added April 2003, after initial release. The `logFatal()`, `logNote()` calls were added to instrument existing error pathsΓÇöpattern of retrofit instrumentation.

3. **macOS special-casing:** The engine assumes macOS will provide native `Cocoa` implementations for dialogs, URLs, etc., keeping SDL code minimal. This reflects the original Marathon codebase's Mac heritage.

4. **Assertion philosophy:** `_alephone_assert()` is fatal and always calls `vhalt()`, not a debug-only check like C's `assert()`. This is appropriate for a shipped game engine where assertions indicate data invariant violations that cannot be recovered.

5. **Widget framework integration:** The in-game dialog uses the same `dialog` + `vertical_placer` + `w_*` widgets as the menus/UI, demonstrating tight integration with the UI subsystem. Text wrapping leverages `get_theme_font()`, so alert appearance is theme-aware.

## Potential Issues

1. **Buffer reuse race condition:** Static `assert_text[256]` is reused across calls. If alerts are triggered concurrently (unlikely in single-threaded game loop, but possible in network/worker threads), the message will be corrupted. No mutex protection.

2. **Text wrapping doesn't handle long words:** If a single word exceeds `MAX_ALERT_WIDTH`, it will display overflowing the dialog boundary. No hyphenation or truncation fallback.

3. **Silent failure on URL launch (Linux):** Fork+execlp for `xdg-open` and `sensible-browser` fails silently if both binaries are missing or unexecutable. Parent `wait()` is called but exit status isn't checked. User won't know the URL didn't open.

4. **Browse callback uninitialized state:** In `browse_callback_proc()`, if `GetCurrentDirectoryW()` fails, `wcwd` is uninitialized, and `SendMessageW()` sends garbage to the dialog. No error check.

5. **A1_NETWORK_STANDALONE_HUB suppresses all alerts:** `#ifndef A1_NETWORK_STANDALONE_HUB` completely disables alerts in hub mode, potentially hiding critical errors. Fatal errors still call `abort()`, but non-fatal alerts are silent.

6. **Message encoding on Windows:** `utf8_to_wide()` is called but not defined in this file. If the conversion fails or returns incomplete data, `MessageBoxW()` may display garbled text. No validation of the conversion result.
