# Subsystem Overview

## Purpose

CSeries is the cross-platform abstraction layer for Aleph One, providing platform-agnostic type definitions, MacOS API emulation, and utilities that shield the rest of the engine from OS-specific details. It serves as the foundational entry point for core types, strings, fonts, colors, dialogs, timing, file paths, and system utilities used throughout the engine.

## Key Files

| File | Role |
|------|------|
| cseries.h | Master include header aggregating all CSeries utilities |
| cstypes.h | Fixed-width integer types, fixed-point arithmetic, platform constants |
| BStream.cpp / BStream.h | Binary stream serialization/deserialization with endianness support |
| byte_swapping.cpp / byte_swapping.h | Byte-order reversal utilities for cross-platform data conversion |
| csstrings.cpp / csstrings.h | String loading from resources, encoding conversion (Mac Roman Γåö UTF-8/UTF-16), printf formatting |
| cspaths_sdl.cpp / cspaths.h | Platform-specific directory path resolution (data, preferences, logs, save files) |
| csalerts_sdl.cpp / csalerts.h | Error dialogs, assertions, fatal error handling, debug utilities |
| csdialogs_sdl.cpp / csdialogs.h | Dialog control manipulation (enable/disable, text updates, selection queries) |
| cscluts_sdl.cpp / cscluts.h | Color lookup table management and global color constants |
| csfonts.h | Font style constants (bold, italic, underline, shadow) and TextSpec structure |
| cspixels.h | Pixel type aliases and bit-depth conversion macros (8/16/32-bit) |
| csmacros.h | Comparison/clamping (MIN/MAX), flag manipulation, bounds-checked array access |
| csmisc_sdl.cpp / csmisc.h | Machine tick counting, thread yielding, blocking input detection |
| mytm_sdl.cpp / mytm.h | SDL-based task scheduling, timer callbacks, mutex-protected concurrency |
| FilmProfile.cpp / FilmProfile.h | Version-specific behavior profiles for film playback compatibility |

## Core Responsibilities

- Define portable fixed-width integer types (`uint8`, `int16`, `uint32`, etc.) and platform constants via SDL2
- Emulate legacy MacOS types (`Rect`, `RGBColor`, `OSErr`) for seamless cross-platform code reuse
- Perform character encoding conversions (Mac Roman Γåö Unicode/UTF-8/UTF-16) for localized string resources
- Convert between big-endian and little-endian byte order for binary data serialization and wad file loading
- Resolve OS-specific file paths for application data, preferences, logs, screenshots, and save games
- Display modal alert dialogs and handle fatal errors with graceful shutdown and logging integration
- Provide dialog widget manipulation functions for UI control state management
- Manage monotonic tick counting with optional debug slow-motion mode and thread-safe sleep/yield primitives
- Schedule periodic timer tasks (simulating Apple's Time Manager) in dedicated SDL threads with mutual exclusion
- Supply utility macros for MIN/MAX, flag testing, bounds checking, and type-safe memory operations
- Load color lookup tables from resource format and maintain global system color registry
- Convert pixel formats between RGB bit-depths (8/16/32-bit packed formats)

## Key Interfaces & Data Flow

**Exposes to the engine:**
- Type system: Fixed-width types, fixed-point arithmetic constants, color and pixel structures
- String utilities: Resource-based string loading with variable expansion, encoding conversion, printf-style formatting
- Path resolution: Application directories (data, preferences, logs, saves) as platform-native paths
- Alert/dialog system: Error dialogs, assertions, user confirmations, scenario selection
- Timing primitives: Monotonic tick counting, sleep operations, input polling with timeout
- Task scheduling: Periodic timer callback registration with drift-free scheduling
- Font specifications: Font ID constants, style flags, file paths for TTF variants
- Utility functions: MIN/MAX/clamp, flag bit manipulation, bounds-checked array access

**Consumes from other subsystems:**
- SDL2 (SDL.h, SDL_endian.h, SDL_net, SDL_thread, SDL_RWops): Cross-platform multimedia primitives
- Platform APIs (Windows: shlobj.h, shellapi.h; POSIX: sys/wait.h, pwd.h): OS-specific directory and shell operations
- Game engine headers: Logging.h, TextStrings.h, Scenario.h, alephversion.h, config.h (autoconf build flags)
- Standard C++ library: `<string>`, `<vector>`, `<chrono>`, `<thread>`, `<atomic>`

## Runtime Role

- **Initialization:** `mytm_initialize()` sets up the global task scheduler mutex; path cache is populated on first access
- **Frame loop:** `machine_tick_count()` called continuously for timing and frame-rate regulation; `sleep_for_machine_ticks()` paces the main loop
- **Throughout session:** Type system used universally; string utilities fetch localized text; tick functions drive physics/rendering intervals
- **Shutdown:** `mytm_cleanup()` reclaims timer task threads and zombie resources
- **On-demand:** Alert dialogs displayed for errors/assertions; paths resolved when accessing save files or preferences; task callbacks executed in separate SDL threads on configured intervals

## Notable Implementation Details

- Conditional compilation guards platform-specific code (`#ifdef __MACOSX__`, `#if defined(__WIN32__)`, `#ifdef ALEPHONE_LITTLE_ENDIAN`)
- Endianness conversion via SDL's `SDL_SwapBE16()`, `SDL_SwapBE32()`, `SDL_SwapBE64()` for portable big-endian binary I/O
- String resource system uses resource ID and index to retrieve localized text; template variables (`$appName$`, `$appVersion$`) expanded via Boost string replacement
- Mac Roman encoding conversion via `MultiByteToWideChar` (Windows) or manual lookup tables (Unix)
- Drift-free timer scheduling compensates for task execution delays to maintain precise period; tasks execute in dedicated SDL threads with mutex-protected mutual exclusion
- Debug mode `TIME_SKEW` constant enables slow-motion playback without modifying call sites
- Pixel extractors do not upscale (5-bit values remain 5-bit); callers responsible for scaling to full 8-bit range if needed
- Caching of resolved paths avoids repeated OS API calls; fallback to safe defaults (e.g., current directory) if OS query fails
- FilmProfile switch case for `FILM_PROFILE_ALEPH_ONE_1_7` missing `break` statement (potential fall-through bug)
