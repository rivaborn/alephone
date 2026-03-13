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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Rect` | struct | Rectangle with `int16` coordinates (top, left, bottom, right); emulated from MacOS |
| `RGBColor` | struct | RGB color with three `uint16` components (red, green, blue) |
| `OSErr` | typedef | MacOS error code type (aliased to `int` on non-macOS platforms) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ALEPHONE_LITTLE_ENDIAN` | macro define | global | Conditional flag set if platform uses little-endian byte order (checked via SDL2) |
| `kFontIDMonaco` | const int | global | Font ID constant for Monaco font (value: 4) |
| `kFontIDCourier` | const int | global | Font ID constant for Courier font (value: 22) |
| `noErr` | const int | global | MacOS error code for success (value: 0) |

## Key Functions / Methods

### PlatformIsLittleEndian
- Signature: `constexpr bool PlatformIsLittleEndian() noexcept`
- Purpose: Query platform byte order at compile-time
- Inputs: None
- Outputs/Return: `true` if platform is little-endian; `false` if big-endian
- Side effects: None (constexpr evaluation)
- Calls: None (macro-based conditional)
- Notes: Evaluated at compile-time; used to configure fixed-point and pixel macros throughout the engine

### MakeRect (overload 1)
- Signature: `constexpr Rect MakeRect(int16 top, int16 left, int16 bottom, int16 right)`
- Purpose: Construct a `Rect` struct from individual coordinate values
- Inputs: Four `int16` coordinate values
- Outputs/Return: Initialized `Rect` struct
- Side effects: None
- Calls: None (inline aggregate initialization)

### MakeRect (overload 2)
- Signature: `constexpr Rect MakeRect(SDL_Rect r)`
- Purpose: Convert SDL2's `SDL_Rect` (x, y, w, h) to engine's `Rect` (top, left, bottom, right)
- Inputs: SDL2 `SDL_Rect` struct
- Outputs/Return: Converted `Rect` struct
- Side effects: None
- Calls: None (inline conversion)
- Notes: Performs coordinate system conversion: SDL's (x, y, w, h) ΓåÆ Aleph's (y, x, y+h, x+w)

## Control Flow Notes
This is a header-only initialization file; no frame-loop integration. Included early in compilation to establish types and macros used throughout the engine. Platform detection (`ALEPHONE_LITTLE_ENDIAN`, SDL endianness) occurs at compile-time.

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
