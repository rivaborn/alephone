# Source_Files/CSeries/BStream.cpp
## File Purpose
Binary stream serialization/deserialization layer providing type-safe input/output for primitive data with endianness support. Designed to replace AStream and wrap standard C++ streambuf for game data I/O.

## Core Responsibilities
- Deserialize primitive types (8ΓÇô64-bit integers, doubles) from input streams
- Serialize primitive types to output streams
- Handle big-endian byte swapping via SDL2
- Track stream read/write positions and bounds
- Enforce serialization bounds with exception-based error handling
- Support method chaining for fluent stream operations

## External Dependencies
- **SDL2/SDL_endian.h** ΓÇô `SDL_SwapBE16()`, `SDL_SwapBE32()`, `SDL_SwapBE64()` for platform-independent endian conversion
- **std::streambuf** ΓÇô underlying buffer management (standard C++)
- **cseries.h** ΓÇô project type definitions (uint8, int8, Uint64, etc.)

# Source_Files/CSeries/BStream.h
## File Purpose
Provides serialization/deserialization classes for reading from and writing to `std::streambuf` objects. Replaces the older AStream interface with support for endianness-aware binary I/O (Big-Endian). Used for persistent game data and resource loading.

## Core Responsibilities
- Wrap `std::streambuf` pointers for abstraction over underlying I/O sources (files, memory, pipes, etc.)
- Provide typed extraction (`operator>>`) and insertion (`operator<<`) for primitive types
- Support Big-Endian serialization via concrete `BIStreamBE` and `BOStreamBE` classes
- Handle 8-bit types (no conversion) and multi-byte types (endianness-aware)
- Query stream position and bounds (`tellg()`, `maxg()`, `tellp()`, `maxp()`)
- Support raw byte operations (`read()`, `write()`, `ignore()`)

## External Dependencies
- `cseries.h` (which includes `cstypes.h` ΓÇö defines `uint8`, `int8`, `int16`, `uint16`, `int32`, `uint32`)
- `<streambuf>` (STL; abstract I/O buffer)
- `std::ios_base::failure` (exception type)

# Source_Files/CSeries/byte_swapping.cpp
## File Purpose
Implements byte-swapping utilities for converting between endianness representations. Enables the Aleph One engine to handle multi-byte data in consistent byte order on little-endian platforms.

## Core Responsibilities
- Reverse byte order for 2-byte (16-bit) field sequences
- Reverse byte order for 4-byte (32-bit) field sequences
- Support in-place memory transformation for bulk data conversion
- Conditionally compiled to be a no-op on big-endian systems

## External Dependencies
- `cseries.h`: Common game engine types and endianness detection
- `byte_swapping.h`: Function declaration and `_bs_field` enum definition
- `uint8`: Fixed-width byte type (from cstypes.h via cseries.h)

# Source_Files/CSeries/byte_swapping.h
## File Purpose
Header file declaring byte-swapping utilities for endianness conversion. Provides conditional compilation that disables byte swapping on little-endian platforms (where no conversion is needed), supporting cross-platform data serialization/deserialization.

## Core Responsibilities
- Define `_bs_field` type for specifying swappable field sizes
- Declare `byte_swap_memory()` function for in-place byte reversal on multi-byte fields
- Provide conditional macro that compiles to no-op on little-endian systems
- Support type-tagged field specifications (2-byte, 4-byte enums)

## External Dependencies
- `<stddef.h>` ΓÇô for `size_t` or similar standard type definitions (though not directly visible in declarations)
- `byte_swap_memory` implementation defined elsewhere (likely CSeries module)
- Conditional flag `ALEPHONE_LITTLE_ENDIAN` set by build configuration

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

## External Dependencies
- Includes: `cstypes.h` (for `int32` typedef)
- Compiler-specific: `__attribute__((noreturn))` for GCC; fallback no-op for other compilers via `NORETURN` macro
- External symbols: all function implementations and the resource system (resid/item pairs) are defined elsewhere

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

# Source_Files/CSeries/cscluts.h
## File Purpose
Defines color lookup table (CLUT) structures and color management for the Aleph One game engine. Provides data structures for representing RGB colors and palettes, along with a function to load color tables from resources and a global system color registry.

## Core Responsibilities
- Define RGB color representation (16-bit per channel)
- Define indexed color table structure (up to 256 colors)
- Provide resource loading for color tables
- Maintain global system color constants (black, white, system palette)
- Support color palette management for rendering

## External Dependencies
- `cstypes.h` ΓÇô for `uint16`, `uint32` integer type definitions
- `LoadedResource` ΓÇô forward declared; defined elsewhere; used for resource loading
- `RGBColor` ΓÇô forward declared; defined elsewhere; likely a struct or typedef for color representation

# Source_Files/CSeries/cscluts_sdl.cpp
## File Purpose
SDL-based implementation of CLUT (Color Look-Up Table) resource handling for the Aleph One game engine. Converts classic Mac CLUT resource format to engine color tables and defines global color constants used throughout the engine.

## Core Responsibilities
- Define global color constants (`rgb_black`, `rgb_white`, `system_colors`)
- Parse Mac binary CLUT resource format from memory
- Convert 16-bit big-endian color components to engine color table format
- Manage endian conversion for cross-platform compatibility

## External Dependencies
- **SDL2**: `SDL_RWops`, `SDL_RWFromMem()`, `SDL_RWseek()`, `SDL_ReadBE16()`, `SDL_RWclose()` ΓÇö memory-based I/O and big-endian integer reading
- **FileHandler.h**: `LoadedResource` class for resource management
- **cseries.h**: `RGBColor` struct definition; pulls in platform-specific macros and SDL headers
- **Implicit dependencies**: `color_table` structure definition, `NUM_SYSTEM_COLORS` constant (defined elsewhere)

# Source_Files/CSeries/csdialogs.h
## File Purpose
Header providing cross-platform dialog and UI control utilities for Aleph One. Originally Mac OS-specific, this was adapted to support SDL and other platforms while maintaining a shared API for dialog manipulation across the codebase.

## Core Responsibilities
- Define dialog abstraction types (`dialog` class, `DialogPtr` typedef)
- Define control state constants (`CONTROL_ACTIVE`, `CONTROL_INACTIVE`)
- Declare functions for manipulating dialog controls (text updates, enable/disable, selection queries)
- Provide platform-agnostic interface to UI operations that differ between Mac OS and SDL implementations

## External Dependencies
- `<string>` ΓÇö standard library (included but not directly used in this header)
- `<vector>` ΓÇö standard library (included but not directly used in this header)
- `dialog` class ΓÇö defined elsewhere in the codebase

# Source_Files/CSeries/csdialogs_sdl.cpp
## File Purpose
Provides compatibility wrapper functions for SDL dialog control manipulation, enabling cross-platform source code between SDL and legacy Mac OS versions. These utilities allow modification and querying of dialog widgets without exposing the underlying SDL widget classes.

## Core Responsibilities
- Enable/disable individual dialog controls dynamically
- Query selection state from dropdown/selection widgets (with 1-based index conversion)
- Update static text widget content after dialog creation

## External Dependencies
- `cseries.h` ΓÇö master header providing platform abstractions, SDL includes, and all CSeries subsystem headers
- `sdl_dialogs.h` ΓÇö `dialog` class definition and dialog system constants (`CONTROL_ACTIVE`, etc. if defined there)
- `sdl_widgets.h` ΓÇö `widget`, `w_select`, `w_static_text` class definitions
- **Defined elsewhere:** constants `CONTROL_ACTIVE`, `NONE`; implementation of `dialog::get_widget_by_id()` and widget methods

# Source_Files/CSeries/cseries.h
## File Purpose
Master include header for the CSeries cross-platform abstraction layer, providing platform-agnostic type definitions, MacOS API emulation, and utilities for the Aleph One game engine (Marathon-based). Serves as the entry point for all engine code needing access to core types, strings, fonts, colors, dialogs, and system utilities.

## Core Responsibilities
- Define platform endianness detection and provide endianness-aware constants
- Provide fixed-width integer type aliases (`uint8`, `int16`, `uint32`, etc.) via SDL2
- Emulate MacOS types (`Rect`, `RGBColor`, `OSErr`) for cross-platform compatibility
- Include all CSeries utility headers in a single convenient location
- Define font ID constants (`kFontIDMonaco`, `kFontIDCourier`)
- Guard and aggregate configuration from autoconf (`config.h`)

## External Dependencies
- **SDL2/SDL.h, SDL2/SDL_endian.h** ΓÇö multimedia library for cross-platform types and endianness detection
- **config.h** ΓÇö autoconf-generated configuration flags (e.g., `HAVE_OPENGL`, feature availability)
- **time.h** ΓÇö for `time_t` type used in `cstypes.h`
- **string** ΓÇö for `std::string` (included but not directly used in this file)
- **CSeries module headers** (included downstream):
  - `cstypes.h` ΓÇö fixed-width types, fixed-point macros, constants
  - `csmacros.h` ΓÇö utility macros (MIN/MAX, bitflags, bounds-checking templates)
  - `cscluts.h` ΓÇö color table utilities
  - `csstrings.h` ΓÇö string manipulation and printf-style functions
  - `csfonts.h` ΓÇö font definitions (`TextSpec`)
  - `cspixels.h` ΓÇö pixel format macros (8/16/32-bit color extraction)
  - `csalerts.h`, `csdialogs.h`, `cspaths.h`, `csmisc.h` ΓÇö declared but content not provided

# Source_Files/CSeries/csfonts.h
## File Purpose
Header defining font styling constants and the TextSpec structure for the Aleph One game engine. Provides bitflags for text styling (bold, italic, underline, shadow) and a unified data structure to encapsulate font metadata and file paths.

## Core Responsibilities
- Define text style constants as bitflags for composable text styling
- Provide TextSpec struct to bundle font configuration (ID, size, style, height adjustment, and font file paths)
- Support TTF font variants (normal, oblique, bold, bold_oblique) for style rendering
- Enable consistent font specification across the engine

## External Dependencies
- `cstypes.h` ΓÇô provides int16, uint16 typedefs (SDL2/SDL_types.h-based)
- `<string>` ΓÇô STL for font file path storage


# Source_Files/CSeries/csmacros.h
## File Purpose
Utility header providing macros and inline functions for the Aleph One game engine. Includes comparison operations, flag bit manipulation, bounds checking, and type-safe memory wrappers.

## Core Responsibilities
- Provide MIN/MAX comparison and value clamping (PIN) with compatibility variants
- Implement flag testing and manipulation for 16-bit and 32-bit fields
- Calculate rectangle dimensions from coordinate structs
- Offer bounds-checked array access via templates (C++ only)
- Provide type-safe memcpy/memset wrappers for object operations (C++ only)

## External Dependencies
- `<string.h>` ΓÇô `memcpy()`, `memset()`
- `FilmProfile.h` ΓÇô `film_profile` global for version-specific behavior; header marked "TERRIBLE" in source comment

# Source_Files/CSeries/csmisc.h
## File Purpose
Header file providing timing and input utility abstractions for the Aleph One game engine. Declares platform-independent functions for machine tick access, thread yielding, and blocking input polls.

## Core Responsibilities
- Declare machine tick timing queries and delays
- Declare cooperative multitasking yield function
- Declare blocking input detection
- Define tick-per-second constant (1000 Hz)

## External Dependencies
- `cstypes.h` ΓÇô type definitions (uint32, uint64_t, bool)
- Implementations not visible in this file; declared externally

# Source_Files/CSeries/csmisc_sdl.cpp
## File Purpose
SDL-based implementation of timing, sleep, and input-wait primitives for the Aleph One game engine. Provides tick-counting utilities with optional debug slow-motion mode (`TIME_SKEW`) and blocking input detection with timeout.

## Core Responsibilities
- Maintain a steady-clock reference epoch for reliable, monotonic millisecond-level tick counting across the game session
- Implement thread-safe sleep operations with frame-timing adjustment support via the `TIME_SKEW` constant
- Provide processor yield to reduce busy-waiting overhead in polling loops
- Poll SDL event queue for user input (mouse, keyboard, controller) with timeout and early exit on match
- Support debug "slow motion" mode via `TIME_SKEW` without changes to call sites

## External Dependencies
- **`<chrono>`** ΓÇô C++11 steady-clock utilities
- **`<thread>`** ΓÇô C++11 threading (`this_thread::sleep_for`, `sleep_until`, `yield`)
- **`"cseries.h"`** ΓÇô Game engine common types and SDL2 includes (defines `uint32`, `uint64_t`, SDL event types)
- **`"Logging.h"`** ΓÇô Logging framework (included but unused in this file)
- **SDL2 event API** (via cseries.h) ΓÇô `SDL_Event`, `SDL_WaitEventTimeout`, `SDL_MOUSEBUTTONDOWN`, `SDL_KEYDOWN`, `SDL_CONTROLLERBUTTONDOWN`

# Source_Files/CSeries/cspaths.h
## File Purpose
Provides a cross-platform interface for accessing OS-specific directory paths (data, preferences, logs, etc.) and application metadata. Part of the Aleph One game engine's core utilities for abstracting filesystem operations.

## Core Responsibilities
- Define enumerated path types for application directories (data, preferences, logs, screenshots, saved games, etc.)
- Provide function to retrieve platform-specific filesystem paths by type
- Supply application name and identifier getters
- Abstract platform-specific path separators and conventions

## External Dependencies
- `cstypes.h` ΓÇô base integer types and constants
- `<string>` ΓÇô C++ standard string library
- Implementation file requires platform-specific includes (e.g., `<windows.h>`, `<unistd.h>`, `<pwd.h>`)

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

## External Dependencies
- **Windows-specific:** `<windows.h>`, `<shlobj.h>` (shell folder APIs); `SHGetFolderPathW`, `GetModuleFileNameW`, `GetUserNameA`.
- **Cross-platform standard:** `<string>`, `getenv()` (POSIX), `strcpy`, `strpbrk`.
- **Internal:** `cstypes.h` (type aliases), `cspaths.h` (enum CSPathType, function declarations), `csstrings.h` (wide_to_utf8 conversion), `alephversion.h` (A1_DISPLAY_NAME macro).
- **Optional:** `confpaths.h` (PKGDATADIR compile-time constant for Linux package installations).

# Source_Files/CSeries/cspixels.h
## File Purpose
Defines pixel type aliases and provides macros for converting between RGB color values and pixel representations at different bit depths (8-bit, 16-bit, 32-bit). This is a utility header for the Aleph One game engine's rendering subsystem.

## Core Responsibilities
- Define typedefs for pixel data at different color depths (`pixel8`, `pixel16`, `pixel32`)
- Provide macros to convert RGB color components (16-bit range) to packed pixel formats
- Provide macros to extract individual R, G, B components from packed pixel formats
- Define constants for color depth limits and component ranges

## External Dependencies
- **`cstypes.h`** ΓÇô Provides integer typedefs (`uint8`, `uint16`, `uint32`) via SDL2 types.
- Guard macros prevent multiple inclusion.

**Notes:**
- Input range for converters is 16-bit (0x0000ΓÇô0xFFFF), but output bit depth is lower per format.
- The 16-bit format uses a 5-5-5 RGB packing (not the more common 5-6-5), leaving 1 unused bit.
- Extractors do *not* upscale; a 5-bit value stays 5-bit, requiring caller to scale if full 8-bit range is needed.

# Source_Files/CSeries/csstrings.cpp
## File Purpose

String handling and localization utilities for the Aleph One game engine. Provides resource-based string retrieval with variable expansion, printf-style formatting, and character encoding conversion (Mac Roman Γåö Unicode Γåö UTF-8). Handles both legacy Mac Roman encoding and modern Unicode/UTF-8 standards.

## Core Responsibilities

- Retrieve localized strings from the TextStrings resource system by resource ID and index
- Perform printf-style string formatting
- Convert between Mac Roman (legacy Mac encoding) and Unicode/UTF-8
- Support wide character (UTF-16) conversion on Windows platforms
- Expand template variables (`$appName$`, `$appVersion$`, `$scenarioName$`, etc.) in strings
- Provide deprecated logging functions (dprintf/fdprintf) that delegate to the modern Logging framework

## External Dependencies

- **TextStrings.h**: `TS_GetCString()`, `TS_CountStrings()` ΓÇô resource-based string storage
- **cspaths.h**: `get_application_name()` ΓÇô application branding
- **alephversion.h**: Version constants (`A1_DISPLAY_VERSION`, `A1_VERSION_STRING`, `A1_DISPLAY_PLATFORM`, `A1_DISPLAY_DATE_VERSION`, `A1_HOMEPAGE_URL`)
- **Logging.h**: `GetCurrentLogger()`, logging levels (`logAnomalyLevel`) ΓÇô modern logging framework (used by dprintf/fdprintf)
- **Scenario.h**: `Scenario::instance()->GetName()`, `GetVersion()` ΓÇô current scenario metadata
- **boost/algorithm/string/replace.hpp**: `boost::replace_all()` ΓÇô template variable substitution
- **Windows API** (Windows only): `MultiByteToWideChar`, `WideCharToMultiByte`, `<wchar.h>`, `<windows.h>`

# Source_Files/CSeries/csstrings.h
## File Purpose
Header file declaring string manipulation and character encoding utility functions for the Aleph One game engine. Provides resource string loading, formatted output (printf-style), character encoding conversions (Mac Roman Γåö Unicode/UTF-8/UTF-16), and application variable expansion.

## Core Responsibilities
- Declare resource string loading from game resource sets
- Declare printf-style formatted output functions (console, debug log, file)
- Declare character encoding conversions (Mac Roman, UTF-8, UTF-16/wide)
- Declare C++ string utility wrappers
- Declare application variable substitution in strings
- Define compiler-specific printf format attributes for type safety

## External Dependencies
- **cstypes.h** ΓÇö defines `uint16` and other fixed-width integer types
- **SDL2/SDL_types.h** ΓÇö provides platform-neutral integer types
- **C++ standard library** ΓÇö `<string>`, `<vector>`

# Source_Files/CSeries/cstypes.h
## File Purpose
Foundational type definitions and macros for the Aleph One game engine. Provides portable integer types with specific bit widths, fixed-point arithmetic support (16.16 format), and common constants. Ensures cross-platform compatibility through conditional compilation.

## Core Responsibilities
- Define fixed-width integer types (uint8, int8, uint16, int16, uint32, int32) via SDL
- Provide fixed-point arithmetic macros and constants (16.16 format)
- Define platform-independent constants (NONE, UNONE, MEG, KILO)
- Supply utility macros for four-character codes (resource identifiers)
- Ensure OpenGL availability fallback for non-OpenGL builds
- Define time-related types (TimeType/time_t)

## External Dependencies
- `<limits.h>` ΓÇö Integer limits
- `<SDL2/SDL_types.h>` ΓÇö Uint8, Sint8, Uint16, Sint16, Uint32, Sint32 type definitions
- `<time.h>` ΓÇö time_t definition
- `config.h` (conditional) ΓÇö Build-time feature flags (HAVE_OPENGL, etc.)

# Source_Files/CSeries/FilmProfile.cpp
## File Purpose
Defines version-specific compatibility and bug-fix configurations for different Marathon and Aleph One engine versions. Provides a mechanism to load the correct behavioral profile based on film/replay version.

## Core Responsibilities
- Declare static FilmProfile instances for eight engine versions (Marathon 2, Marathon Infinity, Aleph One 1.0ΓÇô1.7)
- Initialize a global mutable `film_profile` instance with a default value
- Provide a switch-based loader to activate the appropriate profile configuration

## External Dependencies
- `#include "FilmProfile.h"` ΓÇö defines FilmProfile struct (43 boolean fields) and FilmProfileType enum
- No other includes or external symbols

---

**Note:** Missing `break` statement in the `FILM_PROFILE_ALEPH_ONE_1_7` case (line ~362) will cause unintended behavior if that profile is selected, falling through to undefined state. Recommend adding the missing `break`.

# Source_Files/CSeries/FilmProfile.h
## File Purpose

Defines configuration profiles that control engine behavior during film playback in Aleph One (Marathon engine recreation). Each profile contains boolean flags to enable/disable specific behavior quirks and bug fixes, allowing films recorded with different game versions to play back correctly.

## Core Responsibilities

- Define `FilmProfile` struct with ~45+ boolean flags for version-specific engine behaviors
- Enumerate supported film profile types (Marathon 2, Infinity, Aleph One 1.0ΓÇô1.7, etc.)
- Declare global `film_profile` instance to hold active profile
- Declare `load_film_profile()` function to switch between profiles

## External Dependencies

- None visible in this header (standard C struct/enum definitions only)
- Implementation of `load_film_profile()` likely defined in a `.c` file; callers use this header to access the global `film_profile` instance

# Source_Files/CSeries/mytm.h
## File Purpose
Provides a cross-platform task scheduling and timing system for the Aleph One game engine. Manages emulated "Mac Toolbox" tasks (timed callbacks) and thread-safe synchronization via mutex locks for multi-threaded access.

## Core Responsibilities
- Schedule and execute timed callback functions (`myXTMSetup`, `myTMRemove`)
- Provide mutex-protected access for concurrent operations (e.g., packet listening thread)
- Manage cleanup and reclamation of zombie threads (`myTMCleanup`)
- Initialize the task manager before use (`mytm_initialize`)
- Offer exception-safe RAII-style mutex management for C++ code

## External Dependencies
- **cstypes.h** ΓÇö Type definitions (`int32`, `bool`)
- **Underlying:** Likely pthreads, SDL_thread, or similar for thread/mutex implementation (not visible in header)
- **Clients:** Game loop, networking thread, other subsystems that need timed callbacks

# Source_Files/CSeries/mytm_sdl.cpp
## File Purpose
Implements an SDL-based emulation of Apple's Time Manager API for periodic task scheduling. Provides drift-free timer callbacks for networking code and other subsystems that require scheduled execution. Tasks run in dedicated threads with mutual exclusion to prevent concurrent execution.

## Core Responsibilities
- Initialize and maintain a global mutex protecting timer task execution
- Create periodic timer tasks that execute in independent SDL threads
- Implement drift-free period scheduling (compensate for execution delays)
- Enforce mutual exclusion between timer tasks
- Clean up completed tasks and reclaim thread/memory resources
- Profile task execution timing and deadline misses (DEBUG builds only)
- Boost thread priority of timer tasks relative to main thread

## External Dependencies
- **SDL2:** `SDL_Thread`, `SDL_CreateThread()`, `SDL_WaitThread()`, `SDL_mutex`, `SDL_LockMutex()`, `SDL_UnlockMutex()`, `SDL_GetError()`
- **Timing functions:** `machine_tick_count()`, `sleep_for_machine_ticks()` (defined elsewhere)
- **Priority:** `BoostThreadPriority()` from `thread_priority_sdl.h` (defined elsewhere)
- **Logging:** `logWarning()`, `logAnomaly()`, `logDump()` from `Logging.h`
- **Standard library:** `<atomic>`, `<vector>`


